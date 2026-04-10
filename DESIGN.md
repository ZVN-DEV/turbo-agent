# turbo-agent — Design Sketch

> **This is a forward-looking sketch.** Nothing here compiles yet. Most of it depends on Turbo features that don't exist (generics, traits, async, real HTTP, real JSON, refinement types). The purpose of this document is to lock in the *shape* of the library so the compiler features it depends on can be prioritized correctly.

## Design principles (from the 2026-04-09 architectural decision)

These are non-negotiable invariants. Everything below derives from them.

1. **Agent steps are values, not side effects.** Every tool call, every model reply, every retry is a first-class `AgentEvent` you can pattern-match, store, serialize, and replay.
2. **Schemas are types.** A `tool fn` is not a name in a prompt — it's a typed callable whose schema is **derived from its signature by the compiler**, not hand-written by the user, not runtime-reflected.
3. **Capabilities are declared, not inherited.** An agent is a *bounded* callable with an explicit tool capability set. Declaring `agent R` does not automatically grant it `exec`, `http_get`, or filesystem access.
4. **Costs and budgets are typed.** `Usage`, `Cost`, `TokenCount`, `Latency` are stdlib types with compiler-blessed semantics. Budgets are declared on the agent; exceeding them raises a typed error, not a runtime panic.
5. **Determinism is a mode, not a hope.** Every agent run is either `deterministic` (seeded + cached + replayable) or `live`. Tests default to `deterministic`.
6. **Provider is a trait, not a string.** `model:` resolves to a compile-time-known provider implementing a `Provider` trait. New providers are library code.
7. **Agents compose.** An agent can *be a tool* for another agent without ceremony. Handoffs, supervisors, voting, and broadcast are expressible in existing composition primitives — no bespoke keywords.
8. **Observability is a language feature.** `agent.history`, `agent.trace`, `agent.replay(trace)` are in the library and feel like first-class language features.

## Provider trait

```turbo
trait Provider {
    fn ask(self, req: AgentRequest) -> Result<AgentResponse, ProviderError>
    fn stream(self, req: AgentRequest) -> Stream<AgentEventFrame>
    fn supports_tools(self) -> bool
    fn supports_streaming(self) -> bool
    fn cost_for(self, usage: Usage) -> Cost
}

impl Provider for Anthropic { ... }
impl Provider for OpenAI    { ... }
impl Provider for Gemini    { ... }
impl Provider for Local     { ... }
impl Provider for Mock      { ... }
```

`model: "claude-sonnet-4.5"` resolves to a provider via a compile-time registry populated by `@provider("claude-*")` impl attributes. Unknown prefixes fail the build.

## Tool schemas (derived, not written)

```turbo
/// Fetches a webpage and returns its body.
tool fn http_get(
    /// Fully-qualified URL. Must be http or https.
    url: str,
    /// Request timeout in seconds.
    timeout: int = 30,
) -> Result<str, HttpError> { ... }
```

The compiler builds a `ToolSchema { name, doc, params: [(name, TypeExpr, doc)], returns }` from the signature and doc comments at sema time. Default values become schema defaults. `Result<_, E>` return types become the schema's `error` branch. Non-serializable parameter types fail the build.

## Agents with profiles and capability scoping

```turbo
profile Researcher {
    model: "claude-sonnet-4.5"
    budget: tokens(50_000) & dollars(0.25) & latency(30s)
    tools: { http_get, web_search, read_file }
    deny:  { exec, write_file }
    output: ResearchBrief
    system: "You are a careful researcher. Cite sources."
}

agent Carl(topic: str) : Researcher {
    system += "Topic: {topic}. Prefer primary sources."
}

agent Critic : Researcher {
    tools: { read_file }   // narrower than Researcher
    system: "You are a critic. Flag weak citations."
}
```

- `profile` blocks are reusable capability/model bundles, named and inheritable.
- `agent Carl(topic: str)` parametrizes the agent — values splat into `system`/`tools`/`resources` at instantiation. This is what kills Carl Code's `build_persona_msg` hack and lets you express 25 personas in 25 lines.
- `tools:` and `deny:` are sets with set algebra: `tools: Researcher.tools - { exec }` is valid.
- The compiler enforces that an agent's allowed tools are a subset of its profile's, minus `deny:`.

## Bulk agent declaration (compile-time loop)

```turbo
const PERSONAS: [PersonaSpec; 25] = [ ... ]

for p in PERSONAS {
    agent p.name : Researcher {
        system: p.system_prompt
        tools: p.capability_set
    }
}
```

Compile-time loop over a `const` array, evaluated in sema, producing 25 `AgentDef`s in the module. 25 agents cost 25 lines total. Carl Code's persona system collapses into this.

## AgentEvent and observability

```turbo
enum AgentEvent {
    UserMessage    { text: str, at: Instant },
    ModelMessage   { text: str, usage: Usage, at: Instant },
    ToolCall       { id: ToolCallId, tool: str, args: Json, at: Instant },
    ToolResult     { id: ToolCallId, result: Result<Json, ToolError>, at: Instant },
    Plan           { steps: [str], at: Instant },
    Decision       { chose: str, alternatives: [str], reason: str, at: Instant },
    BudgetExceeded { kind: BudgetKind, limit: f64, used: f64, at: Instant },
}

type Trace = [AgentEvent]

let brief = Carl.ask("Summarize RLHF")
for ev in Carl.history {
    match ev {
        AgentEvent::ToolCall { tool, args, .. } => log("called {tool}({args})"),
        AgentEvent::Decision { chose, reason, .. } => audit(chose, reason),
        _ => {}
    }
}

// Persist + replay
fs.write("carl.trace.json", Carl.history.to_json())
let replayed = Carl.replay(Trace::from_json(fs.read("carl.trace.json")))
```

Replay mode short-circuits provider calls to the event log instead of hitting the wire. Default mode for `@test` functions.

## Costs and budgets as typed runtime concepts

```turbo
struct Usage   { input_tokens: u64, output_tokens: u64, cached_tokens: u64 }
struct Cost    { dollars: f64, source: CostSource }
struct Latency { ms: u64 }

enum Budget {
    Tokens(u64),
    Dollars(f64),
    Latency(Duration),
    And([Budget]),
    Or([Budget]),
}

agent Carl : Researcher {
    budget: tokens(50_000) & dollars(0.25) & latency(30s)
}

match Carl.ask("...") {
    Ok(x) => use(x),
    Err(AgentError::BudgetExceeded { kind: BudgetKind::Dollars, limit, used }) => {
        log("Carl spent ${used}, limit ${limit}"); fallback()
    }
    Err(e) => propagate(e),
}

print(Carl.spent)     // Cost
print(Carl.used)      // Usage
```

Every `Provider` impl is required to return `Usage` alongside text. The library aggregates per-agent. Budget checks run before every outbound call.

## Multi-agent orchestration without new keywords

Agents are tools. An agent whose signature matches a tool is callable by another agent by name.

```turbo
agent Researcher : ResearcherProfile { output: ResearchBrief }
agent Critic     : CriticProfile     { output: Critique }

tool fn ask_researcher(topic: str) -> ResearchBrief { Researcher.ask(topic) }
tool fn ask_critic(brief: ResearchBrief) -> Critique { Critic.ask_structured(brief.to_json()) }

agent Editor : EditorProfile {
    tools: { ask_researcher, ask_critic }
    output: FinalArticle
}
```

Handoffs are function calls. Supervisors are agents whose tools are other agents. Voting is `map` + `reduce` over `[ResearchBrief]`. Broadcasts are `spawn`.

**Sugar (post-v0.1):** Putting an agent name directly in a `tools:` set auto-generates the wrapper:

```turbo
agent Editor : EditorProfile {
    tools: { Researcher, Critic }    // auto-wraps as ask_*
}
```

## Streaming typed structured output

```turbo
struct ResearchBrief {
    title: str where len > 0 && len < 200
    summary: str where len <= 2000
    citations: [Citation] where len >= 1 && len <= 10
}

agent Carl : Researcher {
    output: stream ResearchBrief   // partials emitted as fields fill in
}

for partial in Carl.ask_stream("Summarize transformer arch") {
    match partial {
        Partial::Field("title", v)     => ui.set_title(v),
        Partial::Field("summary", v)   => ui.set_body(v),
        Partial::Field("citations", v) => ui.push_citation(v),
        Partial::Done(full)            => ui.finalize(full),
    }
}
```

`where` clauses are checked as refinement predicates at the model-output boundary. Refinement failure → bounded automatic retry → `StructuredOutputError`. **This streams typed fields, not tokens.** Token streaming is a provider implementation detail.

## Durable execution (post-1.0)

```turbo
@durable
agent Carl : Researcher { ... }

fn main() {
    let brief = Carl.run("Summarize RLHF")   // checkpointed after each event
    // If the process dies mid-run, next invocation resumes from the last event.
}
```

Append-only event log persisted to a `TurboStore` (sqlite/file/redis/pluggable). Tool results memoized by `(tool_name, canonicalized_args_hash)`. `@durable` forbids tools that return `Future` (non-deterministic) types unless they're explicitly marked `@idempotent`.

This is the post-1.0 crown jewel — the Restate pitch, but at the language level. It only works because the `AgentEvent` log from earlier sections is the substrate.

## What's NOT in scope (deliberate exclusions)

- **No vector store / embeddings / RAG plumbing.** Library territory. Will be wrong in 2 years.
- **No `chat` / `conversation` / `LLM` / `memory` keywords.** A conversation is `[AgentEvent]`. A model is `Provider` + name. Memory is `Trace` persisted via `TurboStore`.
- **No baked-in prompting style** (ReAct, ToT, Reflexion). Goes stale fast.
- **No user-overridable tool schemas.** Compiler-derived from signatures, period. The moment we let users hand-write a schema override, we become LangChain-with-better-syntax.
- **No mutable agents** by default. Runtime capability mutation destroys compile-time guarantees. Gated behind `@dynamic_tools` if ever needed.
- **No token streaming as a language concept.** `stream T` is typed structured-output streaming. Tokens are a provider detail.
- **No built-in eval harness, no UI primitive.** Both are opinionated workflow choices that belong in separate tools.

## Implementation phases (when prerequisites land)

- **v0.1 — "Real provider trait + tool calling."** Provider, ToolSchema (derived), real Anthropic/OpenAI tool_use parsing, Usage capture, basic Agent with capability sets.
- **v0.2 — "Agents compose and observe."** profile blocks, set algebra, parametrization, bulk expansion, AgentEvent + history, replay-from-trace.
- **v0.3 — "Budgets and structured output."** Budget types + enforcement, refinement types on output, streaming typed partials.
- **v0.4 — "Agents as tools sugar."** Auto-wrapping agents in `tools:` sets. Multi-provider ensembles.
- **v1.0 — "Stable surface."** Provider trait stabilized. Public ABI for third-party providers. LSP integration (hover on `tool fn` shows derived schema).
- **Post-1.0 — `@durable`, `agent.fork(n).join()`, plan types, multi-model voting.**

This roadmap is intentionally library-shaped, not language-shaped. None of it adds keywords to Turbo.
