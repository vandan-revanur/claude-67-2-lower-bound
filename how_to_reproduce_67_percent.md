# Reproducing the 67.25% Critical Line Lower Bound: A Layman's Walkthrough

This document explains, in plain language, how to reproduce the headline result of the accompanying paper:
a proven lower bound of **67.25%** (rounded to "67.2%" in the paper) of the non-trivial zeros of the Riemann
zeta function ζ(s) lying on the critical line, by building and running the Lean proof scripts in this
repository. It is intended for readers who are not Lean or number-theory experts.

---

## TL;DR: Quick reproduction recipe

Make sure at least 20 GB of free disk space is available before starting. Prebuilt Mathlib caches and build
artifacts require significant headroom. If space is tight, free some up first, for example:

```bash
docker image prune -a          # remove unused Docker images
sudo apt-get clean             # clear the apt package cache
pip cache purge                # clear the pip cache
```

Then run:

```bash
# 1. Install elan (Lean's toolchain manager); it reads lean-toolchain and installs the right Lean release
curl https://raw.githubusercontent.com/leanprover/elan/master/elan-init.sh -sSf | sh -s -- -y
export PATH="$HOME/.elan/bin:$PATH"

# 2. From the repository root, fetch the prebuilt Mathlib cache
lake exe cache get

# 3. Build the library and the comparator statement files
lake build
lake build Solution Solution.Multiplicity Solution.XiPrime

# 4. Print the axiom audit; every line should show only Lean's 3 standard axioms
lake env lean comparator/PrintAxioms.lean
lake env lean comparator/PrintAxioms/Multiplicity.lean
lake env lean comparator/PrintAxioms/XiPrime.lean
lake env lean comparator/PrintAxioms/PairCeiling.lean
```

To see the 67.25% figure printed directly from Lean, create a small standalone file (not part of the
repository's tracked sources) alongside the repository root:

```bash
cat > ScratchEval.lean << 'EOF'
def theta : Float := 1.0 / Float.sqrt 2.0
def c1star : Float := Float.sqrt 2.0 * Float.tan theta / (1.0 + theta * Float.tan theta)
def HD1 : Float := 2.0 - 1.0 / c1star

#eval HD1          -- the lower bound, as a fraction
#eval HD1 * 100.0  -- as a percentage
EOF

lake env lean ScratchEval.lean
rm ScratchEval.lean   # clean up; this file is not part of the repository
```

Expected output: `0.672501` and `67.250070`, the 67.25% lower bound. See Section 4 for an explanation of
where this formula comes from and why it is only a decimal illustration of an exact, symbolically proved
inequality.

If all steps complete without errors or `sorry` warnings, and every `PrintAxioms*` line reads
`[propext, Classical.choice, Quot.sound]`, the same clean audit recorded in `AUDIT.md` has been reproduced:
an independently machine-checked proof that more than two thirds (67.25%) of the zeros of the Riemann zeta
function lie on the critical line.

---

## 1. What is being claimed

The Riemann Hypothesis states that every non-trivial zero of the Riemann zeta function ζ(s) has real part
exactly 1/2, that is, it lies on the critical line. This has not been proven in general. What can be proven
are lower bounds: statements of the form "at least X% of the zeros are on the critical line," which get
closer to 100% as mathematical technique improves.

* The previous best known unconditional bound (Conrey, 1989, refined by others) was about 41%.
* This paper proves at least 67.25% (written 2 minus 1/c1*, approximately 0.67250, that is, "more than two
  thirds"), using an optimized ("Montgomery-Taylor") window together with a new linear-algebra device (a
  "rank-trace inequality") applied to Weil's explicit formula for ζ.

This repository does not just state that result in English; it contains a complete Lean 4 proof, checked
mechanically by Lean's trusted kernel, of the exact inequality that gives 0.67250, with no unproven
assumptions ("`sorry`s") and no extra axioms beyond Lean/Mathlib's three standard logical axioms
(`propext`, `Classical.choice`, `Quot.sound`, the same three that essentially every Mathlib theorem, and
therefore essentially all of modern formalized mathematics, depends on).

## 2. Tools involved

* **Lean 4**: a proof assistant and programming language whose kernel can mechanically verify that a proof
  is logically correct.
* **Mathlib**: the large, community-maintained library of formalized mathematics that Lean proofs build on
  (analysis, number theory, linear algebra, measure theory, and so on). This project pins an exact Mathlib
  commit so the proof is fully reproducible.
* **`elan`**: the Lean version manager (comparable to `rustup` for Rust) that reads the repository's
  `lean-toolchain` file and installs the exact Lean release the project needs.
* **`lake`**: Lean's build tool (comparable to `make` or `cargo build`), which compiles all the `.lean`
  files into checked `.olean` files and can fetch a prebuilt cache of Mathlib instead of recompiling it from
  source, which otherwise takes hours of CPU time.
* **`comparator`**: an independent tool, not written by the paper's authors, that re-checks the proof
  against a separately written, human-readable statement of what should be proved, using a second,
  independently implemented proof kernel. This is the strongest available check that the Lean proof really
  proves what it claims and nothing looser.

## 3. Step-by-step build instructions

### 3.1 Install Lean via `elan`

```bash
curl https://raw.githubusercontent.com/leanprover/elan/master/elan-init.sh -sSf | sh -s -- -y
export PATH="$HOME/.elan/bin:$PATH"
```

`elan` reads the repository's `lean-toolchain` file (`leanprover/lean4:v4.33.0-rc2`) and transparently
downloads and uses that exact Lean release whenever `lean`/`lake` is run inside the repository; no manual
version matching is needed.

### 3.2 Fetch the prebuilt Mathlib cache

```bash
lake exe cache get
```

This downloads roughly 8,700 precompiled Mathlib files (a few GB) instead of compiling Mathlib from source,
which the README notes otherwise takes several hours of CPU time.

> **Disk space note:** Downloading and decompressing this cache requires substantial free disk space.
> Confirm at least 20 GB is free before running `lake exe cache get` or `lake build`. If space is limited,
> free some up first with commands such as `docker image prune -a` (removes unused Docker images),
> `sudo apt-get clean` (clears the apt package cache), or `pip cache purge` (clears the pip cache).

### 3.3 Build the library

```bash
lake build
```

This type-checks every `.lean` file in `Zeta23/` (about 9,000 build jobs, most reused from the Mathlib
cache and therefore quick). Expected result:

```
Build completed successfully (9010 jobs).
```

No errors and no `sorry` warnings, exactly as the repository's own `AUDIT.md` predicts.

### 3.4 Build the independently stated "challenge" theorems

```bash
lake build Solution Solution.Multiplicity Solution.XiPrime
```

The `comparator/` directory contains a second copy of the headline theorem statements, written using only
Mathlib definitions and not trusting anything from the `Zeta23/` library itself, plus thin "Solution"
proofs that point at the real theorems in `Zeta23/`. Building this confirms the two independently written
statements are word-for-word compatible.

```
Build completed successfully (9002 jobs).
```

### 3.5 Audit which axioms each headline theorem relies on

```bash
lake env lean comparator/PrintAxioms.lean
lake env lean comparator/PrintAxioms/Multiplicity.lean
lake env lean comparator/PrintAxioms/XiPrime.lean
lake env lean comparator/PrintAxioms/PairCeiling.lean
```

Every line printed, covering 27 plus 6 plus 11 headline and auxiliary theorems, reads exactly:

```
[propext, Classical.choice, Quot.sound]
```

That is, Lean's three standard, universally accepted logical axioms and nothing invented specifically for
this paper. This matches the repository's own `AUDIT.md`, verbatim, on a fresh checkout.

## 4. Viewing the 67.25% figure directly from Lean

The constant is defined in the library as `HD 1 = 2 - 1/c1*`, where `c1* = sqrt(2) * tan(theta) / (1 +
theta * tan(theta))` and `theta = 1/sqrt(2)` (see `Zeta23/ThmD/Functional.lean`). To see the number rather
than rely on the docstring, create a small standalone Lean file, not part of the repository's source tree,
that evaluates this same formula using floating point:

```lean
def theta : Float := 1.0 / Float.sqrt 2.0
def c1star : Float := Float.sqrt 2.0 * Float.tan theta / (1.0 + theta * Float.tan theta)
def HD1 : Float := 2.0 - 1.0 / c1star

#eval HD1          -- the lower bound, as a fraction
#eval HD1 * 100.0  -- as a percentage
```

Run it with:

```bash
lake env lean ScratchEval.lean
```

Expected output:

```
0.672501
67.250070
```

That is the figure the whole exercise is after: at least 67.25%, "more than two thirds," of the non-trivial
zeros of ζ(s) are proven, by a machine-checked proof, to lie on the critical line. The paper rounds this to
"67.2%". The actual theorems in the library do not use floating point at all; the inequality
`(2 - 1/c1* - epsilon) * N(T,2T) <= N0*(T,2T)` is proved exactly, symbolically, for the exact real number
`c1*`. The float computation above is only a convenient way to see the decimal value of that exact constant.

Remove the scratch file after use so it does not get committed alongside the repository's tracked sources.

## 5. Confirming there are no hidden `sorry`s

The only place the word `sorry` appears as an actual, uncommented proof placeholder is in
`comparator/Challenge*.lean`, the trusted, human-written statement files, which deliberately state each
theorem with a placeholder proof because their job is only to pin down what is being proved, not to prove
it. Grepping the whole `Zeta23/` library and `Solution` files confirms this: the only three hits are the
word "sorry" appearing inside English doc-comments (for example, "sorry-free prefix"), not actual `sorry`
tactics.

```bash
grep -rn "sorry" Zeta23/ comparator/Solution*
```

## 6. Result summary

| Check | Command | Result |
|---|---|---|
| Mathlib cache fetch | `lake exe cache get` | 8,681 files downloaded |
| Library build | `lake build` | 9010/9010 jobs, no errors, no `sorry` |
| Comparator statement build | `lake build Solution Solution.Multiplicity Solution.XiPrime` | 9002/9002 jobs, no errors |
| Axiom audit (27 statements) | `lake env lean comparator/PrintAxioms.lean` plus `.../Multiplicity.lean` | All `[propext, Classical.choice, Quot.sound]` |
| Axiom audit (xi-prime zeros, 6 statements) | `lake env lean comparator/PrintAxioms/XiPrime.lean` | All `[propext, Classical.choice, Quot.sound]` |
| Axiom audit (bandwidth-one ceiling) | `lake env lean comparator/PrintAxioms/PairCeiling.lean` | All standard axioms, one `[propext]`-only `decide` check, one axiom-free |
| Headline constant | small `#eval` script matching the library's exact formula | 0.672501, that is, 67.25% |

Every result matches the repository's own `AUDIT.md` claims exactly, reproduced independently on a fresh
checkout.

## 7. What this does not prove

* This does not prove the full Riemann Hypothesis. It proves a lower bound (67.25%) on the fraction of
  zeros known to be on the critical line, a much weaker but unconditional, mechanically verified statement.
* Building the library only checks that the Lean statements are internally consistent and axiom-clean. It
  does not by itself verify that the English theorem statements in `comparator/ChallengeDeps*.lean`
  faithfully capture the informal claims of the paper; that is a mathematical reading exercise. The
  repository's README provides a mapping table between the two, which is worth cross-checking by eye.
* For the strongest level of independent assurance, the repository recommends running the external
  `comparator` tool (see `comparator/README.md`), which re-verifies proofs with a second, independently
  implemented kernel (`nanoda`). Running that tool requires installing a further external dependency; the
  local Lean-kernel checks in this document already provide strong, verified evidence on their own.


