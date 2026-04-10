# turbo-agent

The official Turbo Lang agent framework — written in pure Turbo source code.

## Status: Placeholder

This repo is an **intentional placeholder**. As of 2026-04-09, the in-compiler `agent { }` block, `tool fn` modifier, `Ty::Agent` type, and LLM provider runtime were removed from the Turbo compiler. They will return as a sidecar library here, but only once the core language has the prerequisites it needs to host a real implementation:

- [ ] Real HTTP client in stdlib (not a `curl` shell-out)
- [ ] Real JSON parser + serializer in stdlib
- [ ] Generics on structs and traits
- [ ] A real trait system with `impl Trait for Type`
- [ ] Async / Futures that compose with I/O
- [ ] Refinement types on struct fields (for structured output validation)

When those land, this repo gets a real implementation. Until then, see [`DESIGN.md`](./DESIGN.md) for the planned API.

## Why a sidecar instead of a language feature

The compiler used to have soft keywords `agent`, `tool`, `resource`, `prompt`, plus a 7-pointer agent struct, hardcoded `.ask` / `.ask_structured` / `.stream` method dispatch in codegen, and an OpenAI/Anthropic runtime that `fork()`-ed `curl` and used `strstr` to scrape responses. None of it parsed real `tool_use` blocks. None of it tracked tokens or cost. The `mock:toolloop` "tool loop" was a string-matching puppet show.

That's bad compiler code, and worse, it was the **wrong** kind of code. Languages that ship 8-year horizons cannot bake in 2025-era domain assumptions — see PHP `mysql_*`, Perl CGI, every framework that went all-in on Web 2.0. Libraries age well; keywords don't.

The bigger insight: **the infrastructure this library needs is exactly what Turbo needs to be a serious general-purpose language anyway.** Real HTTP, real JSON, generics, traits, async. Building those for the sidecar is the same work as making Turbo a better language. So this placeholder is also a forcing function for the right kind of compiler improvements.

The pitch: *"Turbo is so expressive that the official agent framework is pure-Turbo source code."* Same play as Rust + tokio, Go + net/http.

## Planned design (what lives here eventually)

- **`Provider` trait** with impls for Anthropic, OpenAI, Gemini, local, mock
- **`ToolSchema`** derived by the compiler from `tool fn` signatures and doc comments — never hand-written
- **`AgentEvent`** enum, **`Trace`** type, **`agent.history`**, **`agent.replay(trace)`**
- **`Usage`**, **`Cost`**, **`Budget`** types with runtime enforcement and typed `BudgetExceeded` errors
- **`profile` blocks** — reusable capability/model bundles with set algebra (`tools: Researcher.tools - { exec }`)
- **Bulk agent expansion** — `for p in PERSONAS { agent p.name : Profile { ... } }` so 25 personas cost 25 lines
- **Agents-as-tools composition** — supervisors, voting, handoffs are `map`/`reduce`/`spawn` over agents that satisfy the tool signature. No new keywords.
- **Streaming structured output** — `stream T` yields typed `Partial<T>` field-by-field, not raw tokens
- **`@durable` execution** (post-1.0) — Restate-style checkpointing on top of the `AgentEvent` log

## Consumers

- **[Carl Code](https://github.com/ZVN-DEV/carl-code)** (private) — multi-agent CLI tool, currently pinned to Turbo v0.7.2 awaiting this library. Will be the first dogfood consumer when v0.1 ships.

## Versioning

This repo will follow Turbo's version cadence. The first real release will be `turbo-agent v0.1.0`, paired with the Turbo release that lands HTTP + JSON + generics + traits.

## License

MIT (matches Turbo Lang).
