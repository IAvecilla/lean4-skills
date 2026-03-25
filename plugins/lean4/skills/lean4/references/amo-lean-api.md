# AMO-Lean API Reference

AMO-Lean is a verified optimizer that transforms Lean 4 specs into optimized C/Rust code via equality saturation. The source lives at `$AMO_LEAN_ROOT`.

## Core Types

### `AmoLean.Expr α` — Expression AST

The fundamental type. Represents arithmetic expressions over a base type `α`.

```lean
import AmoLean.Basic

inductive Expr (α : Type) where
  | const : α → Expr α           -- Literal constant
  | var : VarId → Expr α         -- Variable (VarId = Nat)
  | add : Expr α → Expr α → Expr α
  | mul : Expr α → Expr α → Expr α
  | pow : Expr α → Nat → Expr α
```

**Building expressions:**
```lean
open AmoLean (Expr)
open AmoLean.Expr (var const add mul pow)

let x := var 0
let poly := add (const 2) (add (mul x (const 3)) (mul x (mul x (const 5))))
-- Represents: 2 + 3x + 5x²
```

### `AmoLean.Expr.denote` — Semantic evaluation

Connects syntax to semantics. Given an environment mapping variables to values:

```lean
def denote [Add α] [Mul α] [Pow α Nat] (env : VarId → α) : Expr α → α
```

Usage: `⟦expr⟧ env` evaluates the expression.

## Optimization Pipeline

### Pipeline 1: Scalar E-Graph Optimization

```
Expr Int → optimizeExpr → (optimized Expr Int, OptStats) → exprToC → C code
```

**Key imports:**
```lean
import AmoLean.Basic
import AmoLean.CodeGen
import AmoLean.EGraph.Optimize

open AmoLean (Expr)
open AmoLean.Expr (var const add mul pow simplify exprCost defaultCostModel)
open AmoLean.EGraph.Optimize (optimizeExpr OptStats countOps countNodes optPercentage foldConstants OptConfig)
```

> **Note:** `AmoLean.EGraph.Optimize` is the legacy module. For new code, prefer
> `AmoLean.EGraph.Verified.Optimize` which provides the same API backed by
> a formally verified engine. Utility types (`OptConfig`, `OptStats`,
> `foldConstants`, `countOps`) are shared by both.

**`optimizeExpr`** — Run equality saturation on an expression:
```lean
-- With default config
let (optimized, stats) := optimizeExpr myExpr

-- With custom config
let config : OptConfig := { maxIterations := 30, maxNodes := 1000, maxClasses := 500, safeOnly := false }
let (optimized, stats) := optimizeExpr myExpr config
```

**`OptConfig`** — Saturation parameters:
- `maxIterations` — Max e-graph saturation iterations (default: 10)
- `maxNodes` — Max e-graph nodes before stopping
- `maxClasses` — Max equivalence classes
- `safeOnly` — If true, only apply reducing rules (no structural/expanding)

**`OptStats`** — Optimization statistics:
- `opsBefore` / `opsAfter` — Operation counts
- `originalSize` / `optimizedSize` — AST node counts
- `iterations` — Saturation iterations used
- `egraphNodes` / `egraphClasses` — E-graph size

### Pipeline 2: Matrix → Sigma-SPL → C/Rust

For matrix-level optimizations (NTT, FRI, Poseidon):

```
MatExpr → SigmaExpr → ExpandedSigma → C or Rust code
```

**Key imports:**
```lean
import AmoLean.Sigma.Basic
import AmoLean.Sigma.Expand
import AmoLean.Sigma.CodeGen          -- C backend
import AmoLean.Backends.Rust          -- Rust backend
```

## Code Generation

### C Code Generation

```lean
import AmoLean.CodeGen

-- Generate a C function from an expression
let cCode := AmoLean.exprToC "function_name" ["x", "y"] myExpr
-- Returns a String containing the C function

-- For matrix pipeline:
open AmoLean.Sigma.CodeGen (expandedSigmaToC generateFunction generateCFile matExprToC)
```

### Rust Code Generation

```lean
import AmoLean.Backends.Rust

-- Generates Rust with generic NttField trait
-- Supports: Risc0 (BabyBear), SP1, Plonky3 (Goldilocks/BabyBear)
```

### Other Backends

- `AmoLean.Backends.C_AVX512` — C with AVX-512 SIMD intrinsics
- `AmoLean.Backends.CUDA` — CUDA kernel generation

## Verified Pipeline

For formally verified optimization (recommended for production):

```lean
import AmoLean.EGraph.Verified.Optimize
```

This provides the same API as `AmoLean.EGraph.Optimize` but backed by a formally verified e-graph engine where every rewrite step has a Lean proof of semantic equivalence.

## Correctness Proofs

Located in `AmoLean/Correctness.lean`. Key theorem:

```lean
theorem denote_preserved_by_rule (rule : RewriteRule α) (e e' : Expr α) (env : VarId → α)
    (h : rule e = some e')
    (h_sound : ∀ x y, rule x = some y → denote env x = denote env y) :
    denote env e = denote env e'
```

Every rewrite rule has an individual soundness proof (e.g., `rule_add_zero_right_sound`).

## Domain-Specific Modules

- `AmoLean.NTT` — Number Theoretic Transform (Cooley-Tukey, butterfly, bit-reverse)
- `AmoLean.FRI` — Fast Reed-Solomon IOP (fold, query, verification)
- `AmoLean.Protocols.Poseidon` — Poseidon hash function
- `AmoLean.Plonky3` — AIR constraint optimization
- `AmoLean.Field.*` — Finite field implementations (BabyBear, Goldilocks, Mersenne31)

## Lakefile Dependencies

To use AMO-Lean in your project:

```lean
-- lakefile.lean
require «amo-lean» from git
  "https://github.com/lambdaclass/amo-lean.git"

-- AMO-Lean depends on mathlib v4.26.0 and requires lean4 v4.26.0
-- Your lean-toolchain must match: leanprover/lean4:v4.26.0
```
