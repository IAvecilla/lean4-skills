---
name: autooptimize
description: Autonomous AMO-Lean optimization — end-to-end spec-to-verified-code without interactive guidance
user_invocable: true
---

# Lean4 AutoOptimize (AMO-Lean)

Autonomous optimization using AMO-Lean's equality saturation engine. Runs the full pipeline from Lean 4 spec to verified optimized C or Rust code without interactive guidance.

## Usage

```
/lean4:autooptimize                              # Optimize all definitions
/lean4:autooptimize AmoLeanDemo.lean             # Optimize spec in a specific file
/lean4:autooptimize AmoLeanDemo.lean --target=C  # Target C output
```

## Inputs

| Arg | Required | Default | Description |
|-----|----------|---------|-------------|
| scope | No | all | Specific file or definition to optimize |
| --target | No | C | Output language: `C` or `Rust` |

## Environment

| Variable | Description |
|----------|-------------|
| `AMO_LEAN_ROOT` | Path to the AMO-Lean repository |
| `OPTISAT_ROOT` | Path to OptiSat (verified e-graph engine) |
| `LEAN_PROJECT_PATH` | Path to the active Lean project root |

## Autonomous Workflow

Run all phases without asking — only stop if truly blocked.

1. **Read spec** — identify definitions to optimize, understand the algebraic domain
2. **Setup** — add AMO-Lean/OptiSat deps to lakefile, `lake build`
3. **Rewrite rules** — discover identities from Mathlib, prove each one in Lean
4. **Equality saturation** — feed spec + rules into the e-graph, extract optimal form
5. **Code generation** — emit C/Rust, verify full proof chain with `lake build`
6. **Report** — summarize transformations, output files, and any limitations

## Recovery

- If `lake build` fails: read error, fix, retry
- If a proof is stuck: try `exact?`, `apply?`, decompose into smaller lemmas
- If saturation times out: reduce rule set or increase iteration limit
- If truly blocked: explain the blocker clearly and stop

## Rules

- Work autonomously — do not ask unless truly stuck
- Every rewrite rule must have a Lean proof — no `sorry`, no `axiom`
- Use `lake build` frequently
- Use the Lean LSP MCP for goal inspection
- Explain blockers clearly if optimization cannot proceed
