# AMO-Lean Usage Patterns

Common patterns for using AMO-Lean in optimization workflows.

## Pattern 1: Scalar Polynomial Optimization

The simplest pipeline. Encode a polynomial as `Expr Int`, optimize, emit C.

```lean
import AmoLean.Basic
import AmoLean.CodeGen
import AmoLean.EGraph.Verified.Optimize  -- Use verified pipeline (preferred)
-- import AmoLean.EGraph.Optimize        -- Legacy alternative (deprecated)

open AmoLean (Expr)
open AmoLean.Expr (var const add mul pow simplify exprCost defaultCostModel)
open AmoLean.EGraph.Optimize (optimizeExpr OptConfig)

-- 1. Encode your spec
def myPoly : Expr Int :=
  let x := var 0
  add (const 2) (add (mul x (const 3)) (mul x (mul x (const 5))))

-- 2. Optimize (enable structural rules for Horner)
def config : OptConfig :=
  { maxIterations := 30, maxNodes := 1000, maxClasses := 500, safeOnly := false }

def result := optimizeExpr myPoly config

-- 3. Generate C
def cCode := AmoLean.exprToC "my_poly" ["x"] result.1
```

## Pattern 2: NTT/Butterfly Optimization

For cryptographic NTT with domain-specific modules:

```lean
import AmoLean.NTT.Spec
import AmoLean.NTT.CooleyTukey
import AmoLean.NTT.Butterfly

-- AMO-Lean provides pre-built NTT specs and their optimized decompositions
-- See AmoLean/NTT/ for Cooley-Tukey, bit-reverse, twiddle factor verification
```

## Pattern 3: Matrix Pipeline (Sigma-SPL)

For matrix-level transformations following the SPIRAL approach:

```lean
import AmoLean.Sigma.Basic
import AmoLean.Sigma.Expand
import AmoLean.Sigma.CodeGen
import AmoLean.Backends.Rust

open AmoLean.Sigma (lowerFresh expandSigmaExpr)
open AmoLean.Sigma.CodeGen (expandedSigmaToC generateFunction matExprToC)

-- MatExpr → SigmaExpr → ExpandedSigma → C/Rust
-- This pipeline handles: tensor products, stride permutations, gather/scatter
```

## Pattern 4: Verified Pipeline

For production use where you need formal guarantees:

```lean
import AmoLean.EGraph.Verified.Optimize

-- Same API as AmoLean.EGraph.Optimize but every step is proven
-- The type system enforces that rules have soundness proofs
```

## Lakefile Setup

```lean
-- lakefile.lean
import Lake
open Lake DSL

package «my-project» where
  leanOptions := #[⟨`autoImplicit, false⟩]

require «amo-lean» from git
  "https://github.com/lambdaclass/amo-lean.git"

-- OptiSat is optional — only needed if you want direct e-graph access
require optisat from git
  "https://github.com/lambdaclass/optisat_lean.git"

@[default_target]
lean_lib «MyProject» where
  srcDir := "."
```

**Important:** AMO-Lean requires Lean v4.26.0. Set your `lean-toolchain`:
```
leanprover/lean4:v4.26.0
```

## Workflow: From Spec to Verified C Code

1. **Write your spec** as a plain Lean definition (e.g., `polyEval`)
2. **Prove correctness** of the spec (e.g., `horner_eq_poly`)
3. **Encode** the spec as `AmoLean.Expr Int`
4. **Run** `optimizeExpr` with `safeOnly := false` for full rewrite power
5. **Generate** C via `AmoLean.exprToC` or Rust via the Rust backend
6. **Verify** with `lake build` — all proofs must compile with no `sorry`

## Available Backends

| Backend | Import | Output |
|---------|--------|--------|
| C (scalar) | `AmoLean.CodeGen` | Standard C with `int64_t` |
| C (AVX-512) | `AmoLean.Backends.C_AVX512` | C with SIMD intrinsics |
| Rust | `AmoLean.Backends.Rust` | Rust with generic `NttField` trait |
| CUDA | `AmoLean.Backends.CUDA` | CUDA kernels |

## Domain Modules

| Module | What it provides |
|--------|-----------------|
| `AmoLean.NTT` | NTT specs, Cooley-Tukey, butterfly, bit-reverse |
| `AmoLean.FRI` | FRI fold, query, prover/verifier, Merkle tree |
| `AmoLean.Protocols.Poseidon` | Poseidon hash, S-box, MDS matrix |
| `AmoLean.Plonky3` | AIR constraint optimization |
| `AmoLean.Field.BabyBear` | BabyBear field arithmetic |
| `AmoLean.Field.Goldilocks` | Goldilocks field arithmetic |
| `AmoLean.Field.Mersenne31` | Mersenne-31 field arithmetic |
