---
name: optimize
description: Interactive AMO-Lean optimization — transform a Lean 4 spec into verified optimized C or Rust via equality saturation
user_invocable: true
---

# Lean4 Optimize (AMO-Lean)

Interactive, step-by-step optimization using AMO-Lean's equality saturation engine. Transforms a Lean 4 mathematical specification into formally verified optimized C or Rust code.

## Usage

```
/lean4:optimize                              # Start guided optimization session
/lean4:optimize AmoLeanDemo.lean             # Optimize spec in a specific file
/lean4:optimize AmoLeanDemo.lean --target=C  # Target C output
/lean4:optimize AmoLeanDemo.lean --target=Rust  # Target Rust output
```

## Inputs

| Arg | Required | Default | Description |
|-----|----------|---------|-------------|
| scope | No | all | Specific file or definition to optimize |
| --target | No | C | Output language: `C` or `Rust` |

## Environment

The following environment variables are set by Gauss when this workflow runs:

| Variable | Description |
|----------|-------------|
| `AMO_LEAN_ROOT` | Path to the AMO-Lean repository (equality saturation framework) |
| `OPTISAT_ROOT` | Path to OptiSat (verified e-graph engine in Lean 4) |
| `LEAN_PROJECT_PATH` | Path to the active Lean project root |

## Workflow

### Phase 1: Read the spec

1. Read the target file(s) and identify the definition(s) to optimize
2. Use the Lean LSP MCP server to check types and understand the algebraic domain
3. Confirm the spec with the user before proceeding

### Phase 2: Set up AMO-Lean

1. Read `$AMO_LEAN_ROOT` to understand the AMO-Lean API
2. Read `$OPTISAT_ROOT` to understand the OptiSat e-graph API
3. Add necessary dependencies to `lakefile.lean` if not present
4. Run `lake build` to verify the setup

### Phase 3: Define rewrite rules

1. Identify algebraic identities that apply to the spec's domain
2. Write each identity as a Lean 4 theorem with a formal proof
3. **Every rule must be proved — no `sorry`, no `axiom`**
4. Register rules with the OptiSat e-graph engine
5. Ask the user to review the rules before proceeding

### Phase 4: Run equality saturation

1. Feed the spec and rules into the equality saturation pipeline
2. The e-graph explores equivalent representations
3. A cost model selects the optimal form (minimize ops, enable SIMD, etc.)
4. Present the extracted optimal form to the user

### Phase 5: Generate verified output

1. Emit optimized C or Rust code from the best extraction
2. Each transformation step has a Lean proof of semantic equivalence
3. Run `lake build` to verify the full proof chain
4. Present the output code and explain key transformations

## Rules

- Always read and understand the spec before writing rules
- Every rewrite rule must have a Lean proof — never use `sorry`
- Use `lake build` after each major step to catch issues early
- Use the Lean LSP MCP server for type-checking and goal inspection
- Ask the user before proceeding to the next phase
- If a proof is stuck, try searching Mathlib with `exact?` or `apply?`
