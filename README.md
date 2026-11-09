# DUO

*Diagnose, Untangle, Optimize.* A Julia-based companion tool for the [Uno](https://github.com/cvanaret/Uno) nonlinear optimization solver.

> **Status: early development.** The architecture and verb contract are designed (see [`docs/architecture.md`](docs/architecture.md)) and the repository is scaffolded, but most source files are still stubs. Pre-1.0; not yet usable as a tool.

---

## The problem

Modern nonlinear optimization solvers like Uno expose dozens of options, presets, and tuning knobs, and assume the user can read a Jacobian, recognize a poorly scaled constraint, and tell the difference between an infeasible model and one that is merely badly posed. Domain scientists, the chemists, physicists, control engineers, and energy modelers who actually write the models, are not optimization specialists. When their solve fails or stalls, they typically have no way to know whether the problem is in their formulation, in their scaling, in the solver options they picked, or in the solver itself.

The IDAES group at the U.S. Department of Energy documented this gap directly: in their user-experience study of equation-oriented modelers, *no single modeler was aware of all the available debugging tools*, and the most common diagnostic workflow was "ask a colleague" (Lee et al., 2024). The follow-up case study showed that even with a comprehensive diagnostic toolbox in hand, locating a single unscaled constraint in an industrial flowsheet can take an experienced modeler through three sub-models and a half-dozen source files (Allan et al., 2024).

DUO is positioned against that gap. It surrounds Uno with preprocessing, diagnostics, autotuning, and narration so that a non-specialist can hand it a model, hear what is wrong with it, accept the obvious fixes, and get a reliable solve, without ever picking between `mu_strategy=monotone` and `adaptive` from a manual.

## What DUO does

The name doubles as the workflow: **D**iagnose, **U**ntangle, **O**ptimize.

- **Diagnose.** When Uno reports infeasible, unbounded, iteration-limit, or failure, DUO explains which constraints are likely responsible and why the solve struggled. Methods include l1-relaxation with IIS deflation for infeasibility, and Jacobian/Hessian conditioning analysis for ill-posedness. Reports are structured data; an optional LLM narrator turns them into prose in the user's domain language.
- **Untangle.** Variable, constraint, and objective scaling (Ruiz, gradient-balanced, equilibration) and algebraic constraint reformulation (`1/x = y` → `x·y = 1`, `log(exp(x))` → `x`, `expm1`/`log1p` substitutions). Every rewrite is symbolically validated before application, so an LLM can propose a transformation but cannot corrupt the model.
- **Optimize.** Autotuning over Uno's option space via Bayesian optimization (with optional LLM-suggested priors), and an interactive iteration stepper, both a passive trace recorder and a blocking-callback REPL, that lets users watch a solve evolve one iterate at a time.

DUO is built **on top of Uno**, not into it. It consumes Uno via the existing `UnoSolver.jl` MathOptInterface wrapper. No fork, no patch, no source modification. Anything DUO needs from Uno that isn't already exposed is tracked as an upstream PR for a later version.

## Three ways to reach it

The same verb surface (`load`, `scale`, `diagnose`, `solve`, `trace`, `step`, `tune`, `advise`, `narrate`) is exposed through three interfaces:

```bash
# CLI
duo diagnose model.nl --json
duo scale model.nl --method ruiz --out scaled.jld2
duo solve scaled.jld2 --preset ipopt
```

```julia
# Julia library
using DUO
ir = DUO.load("model.nl")
(scaled, _) = DUO.scale(ir; method = :ruiz)
DUO.solve(scaled; preset = "ipopt")
```

```jsonc
// MCP server (consumed by Claude Code and other MCP clients)
{ "duo": { "command": "julia",
           "args": ["--project=/path/to/DUO/interfaces/mcp",
                    "/path/to/DUO/interfaces/mcp/src/server.jl"] } }
```

Setup details are in [`docs/install.md`](docs/install.md).

## Design principles

Five decisions shape the rest of the architecture:

1. **Built on top of Uno, not into it.** Preprocessing happens before Uno sees the model; postprocessing happens after.
2. **Stateless core, stateful adapters.** The core is a collection of pure verbs of the shape `verb(ProblemIR, options) -> Result`. Session state lives in the CLI's disk files or the MCP server's in-memory registry.
3. **Single verb contract.** Every user-facing action is declared once in `src/verbs.jl`. CLI subcommands and MCP tools are thin renderings of these verbs; they never reimplement logic.
4. **LLMs in the human-facing and expertise-priors layers only.** Numerical work goes to Symbolics/JuMP/Uno. LLMs propose, narrate, and advise. Every LLM-touched modification to the model passes a symbolic validation gate.
5. **MCP-first for LLM features.** The driving LLM (Claude Code) does the reasoning, using DUO's verbs as facts and operations. Internal LLM calls in DUO's own code are reserved for CLI and batch modes where no driving LLM is present.

The full design rationale, technology choices considered and rejected, and the Uno integration matrix are in [`docs/architecture.md`](docs/architecture.md).

## Documentation

- [`docs/architecture.md`](docs/architecture.md) — load-bearing design decisions, technology stack, repository layout
- [`docs/verbs.md`](docs/verbs.md) — the verb contract reference
- [`docs/install.md`](docs/install.md) — CLI and MCP setup
- [`docs/mcp.md`](docs/mcp.md) — MCP integration details
- [`docs/claude_code.md`](docs/claude_code.md) — shipped Claude Code skills
- [`docs/llm_features.md`](docs/llm_features.md) — when each LLM feature fires and what it does

A curated knowledge base of background sources (papers on EO diagnostics, Uno internals, IDAES toolbox case studies) lives in [`.crucible/wiki/`](.crucible/wiki/MANIFEST.md); see the manifest for the topic index.

## License

License TBD; intended to match Uno (MIT) before public release.
