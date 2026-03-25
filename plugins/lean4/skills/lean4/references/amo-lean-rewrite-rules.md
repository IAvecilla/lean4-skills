# AMO-Lean Rewrite Rules Reference

Rewrite rules are the core of AMO-Lean's equality saturation. Each rule is a pattern match on an `Expr` that produces an equivalent `Expr`. Every rule has a formal Lean proof of semantic preservation.

## Built-in Rules

### Identity Rules (REDUCING — always safe)

| Rule | Pattern | Result | Cost Delta |
|------|---------|--------|------------|
| `addZeroRight` | `a + 0` | `a` | -1 |
| `addZeroLeft` | `0 + a` | `a` | -1 |
| `mulOneRight` | `a * 1` | `a` | -1 |
| `mulOneLeft` | `1 * a` | `a` | -1 |

### Zero Rules (REDUCING — eliminates subtrees)

| Rule | Pattern | Result | Cost Delta |
|------|---------|--------|------------|
| `mulZeroRight` | `a * 0` | `0` | -10 |
| `mulZeroLeft` | `0 * a` | `0` | -10 |

### Power Rules (REDUCING)

| Rule | Pattern | Result | Cost Delta |
|------|---------|--------|------------|
| `powZero` | `a^0` | `1` | -5 |
| `powOne` | `a^1` | `a` | -1 |
| `zeroPow` | `0^n` | `0` | -1 |
| `onePow` | `1^n` | `1` | -2 |

### Factoring (STRUCTURAL — key for Horner discovery)

| Rule | Pattern | Result | Cost Delta |
|------|---------|--------|------------|
| `factorLeft` | `a*b + a*c` | `a*(b+c)` | -1 |
| `factorRight` | `b*a + c*a` | `(b+c)*a` | -1 |

### Distribution (EXPANDING — use with caution)

| Rule | Pattern | Result | Cost Delta |
|------|---------|--------|------------|
| `distribLeft` | `a*(b+c)` | `a*b + a*c` | +1 |

### Constant Folding

`foldConstants` evaluates `const a OP const b` at compile time.

## Rule Directions

```lean
inductive RuleDirection where
  | reducing    -- Guaranteed to reduce operation count
  | structural  -- May not reduce count, but normalizes form
  | expanding   -- May increase count (use carefully)
```

- `safeOnly := true` in `OptConfig` restricts to reducing rules only
- `safeOnly := false` enables structural rules (factoring, distribution) which are needed for Horner, NTT butterfly, etc.

## Defining Custom Rules

Rules use the `OptRule` structure:

```lean
structure OptRule where
  name : String
  lhs : Pattern
  rhs : Pattern
  direction : RuleDirection
  costDelta : Int
```

### Pattern Language

```lean
inductive Pattern where
  | patVar : Nat → Pattern        -- Matches any expression
  | const : Int → Pattern         -- Matches a specific constant
  | add : Pattern → Pattern → Pattern
  | mul : Pattern → Pattern → Pattern
  | pow : Pattern → Nat → Pattern
```

### Example: Custom factoring rule

```lean
def myFactorRule : OptRule := {
  name := "my_factor"
  lhs := .add (.mul (.patVar 0) (.patVar 1)) (.mul (.patVar 0) (.patVar 2))
  rhs := .mul (.patVar 0) (.add (.patVar 1) (.patVar 2))
  direction := .reducing
  costDelta := -1
}
```

## Correctness Requirements

Every rule must have a soundness proof in `AmoLean/Correctness.lean`:

```lean
theorem my_rule_sound :
    ∀ e e' : Expr α, myRule e = some e' →
    denote env e = denote env e'
```

The verified pipeline (`AmoLean.EGraph.Verified.Optimize`) enforces this at the type level — you cannot register a rule without its proof.

## Common Optimization Patterns

### Polynomial → Horner form
Rules needed: `factorLeft` + identity rules + constant folding.
The e-graph discovers `a₀ + x*(a₁ + x*(a₂ + ...))` automatically.

### NTT Butterfly simplification
Rules needed: `mulOneRight` + `mulZeroRight` + constant folding.
Eliminates trivial twiddle factor multiplications.

### Poseidon S-box optimization
Rules needed: power rules + constant folding.
Reduces `x^5` chain from 4 muls to 3 via `t=x²; t²*x`.

### FRI Fold specialization
Rules needed: `mulZeroRight` + `addZeroRight`.
When `alpha=0`, eliminates the entire fold computation.
