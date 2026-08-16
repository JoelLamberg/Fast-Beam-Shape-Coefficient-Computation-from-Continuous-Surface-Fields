```
# Fast surface-field → Mie scattering (rewrite)

A clean, fast, overflow-safe rewrite of the MATLAB code behind

> J. Lamberg, L. Lamberg, Z. Taylor, *"Fast beam shape coefficient computation
> from continuous surface fields,"* **Optics Letters 50, 4554 (2025).**

It computes the 
```
```
**beam shape coefficients (BSCs)** of an electromagnetic field

```
```
defined on a compact surface, then the **incident, scattered and internal
```
```


```
```
fields** for a homogeneous Mie sphere — in one continuous surface integral plus
one vector-spherical-harmonic (VSH) expansion.
```
```


```
```

## Files
```
```


```
```

| File | Role |
```
```


```
```
|---|---|
```
```


```
```
| 
```
```
`mieFromSurfaceField.m` | **the whole method** (one function + local helpers): angular spectrum → BSCs → Mie coeffs → fields |

```
```
| 
```
```
`run_CVB.m` | driver: azimuthal cylindrical vector beam (needs `CVB_azimuthal_804.mat`) |

```
```
| `run_Axicon.m` | driver: Bessel–Gauss / axicon beam (needs `AxiconParameters2025New.mat`) |
| `test_planewave_mie.m` | self-test (no data needed) — run this first |

It is 
```
```
**self-contained**: the original helpers `shiftGB`, `PIETAOAll`, `fact`

```
```
are 
```
```
**not** required (they are reimplemented internally, better).

```
```

## Quick start

```matlab
```
```


```
```
test_planewave_mie % should print SELF-TEST PASSED
run_CVB % CVB figure
```
```


```
```
run_Axicon % axicon figure
```
```
```


```
```

## What changed vs. the original
```
```


```
```

**Correctness / robustness**
-
```
```
 **No factorial / Legendre overflow.** Fully-normalized associated Legendre

```
```
recurrence (Holmes–Featherstone) ⇒ the 
```
```
`10^(-m)` scaling and the

```
```
`(isnan)=0` zeroing are gone; the normalization collapses to
`D̄(m,n) = (2−δ_{m0})/(4n(n+1))` — no factorials at all. Stable to large `N`.
-
```
```
 **Bug fixed:** in the original `Axicon.txt`, `dOmega = 0.000117779/length(1800)`

```
```
has 
```
```
`length(1800)==1` (a no-op). Here the surface element is the real

```
```
`Area/Ns`
```
```
.

```
```
-
```
```
 Spherical Bessel via stable recurrences; internal-field logarithmic

```
```
derivative `D_n(mx)` by downward recurrence.
- **Scattered-field Hankel blow-up fixed (critical at large `N`).** The
scattered field `Σ Fₙ·hₙ(kr)` is now summed only up to the Wiscombe Mie order
`nS = ⌈x + 4x^{1/3} + 2⌉`
```
```
 (`x = ka`); the internal sum up to the corresponding

```
```
order for 
```
```
`mx`. Past that order the true `aₙ,bₙ,cₙ,dₙ` have decayed to the

```
```
roundoff floor (`~1e-16`, flat in `n`) while `hₙ(kr)` grows like `10^{58}` for
`n ≫ kr`
```
```
 — their product is pure garbage (`~10^{43}`) that otherwise swamps the

```
```
field near the sphere. The incident sum keeps all 
```
```
`N` modes (`jₙ` is bounded).

```
```
Verified end-to-end: 
```
```
`|Escat|` near `r=a` drops from `6e39` → `4e-13` for

```
```
`mr=1`
```
```
 (correct ≈0), with the correct physical scale for a real scatterer.

```
```

**Speed**
```
```


```
```
- Direction integral on a **Gauss–Legendre(θ) × uniform(φ)** grid (~`2N²` nodes)
instead of ~`16N²` Fibonacci nodes.
-
```
```
 The **φ-integral is done by FFT** (all `m` at once).

```
```
-
```
```
 The surface sum is **vectorized (chunked)** over source points.

```
```
-
```
```
 Net: the dominant BSC stage drops from `O(N⁴)` to ~`O(N³)`.

```
```

## Optional parallelism (`opt.useParallel`)

Stage 5 (field evaluation) — the dominant cost for large evaluation grids — can
```
```


```
```
run on a 
```
```
`parpool` via `parfor`. Set `opt.useParallel = true` (both drivers do).

```
```
It requires the Parallel Computing Toolbox and 
```
```
**degrades gracefully to serial**

```
```
when the toolbox is absent or the flag is false (idiom 
```
```
`parfor (jc, maxW)` with

```
```
`maxW = 0`). Stages 1–2 are left **serial on purpose**: their work is already
multithreaded BLAS, which single-threaded 
```
```
`parfor` workers would only surrender.

```
```

**Benchmark**
```
```
 (`run_Axicon`, `P = 250 000`, `N = 140`, 16 cores):

```
```

| Stage 5 | time | speedup |
```
```


```
```
|---|---|---|
| serial | 126.3 s | — |
| parallel (16 workers) | 31.6 s | 
```
```
**4.0×** |

```
```

Sublinear because the serial baseline already uses multithreaded BLAS and Stage 5
is partly memory-bandwidth-bound — 4× is the real, worthwhile gain. Note a
```
```


```
```
one-time 
```
```
`parpool` startup (~19 s) that is amortized across runs in a session

```
```
(the pool persists). The parallel path also uses 
```
```
**smaller per-chunk memory**

```
```
(half the chunk-size budget, and transient arrays live in the worker processes,
```
```


```
```
so the main MATLAB process stays lean). For tiny/one-off grids, leave it `false`.

## Validation (done in Python, machine precision)

| Piece | Check | Result |
|---|---|---|
| spherical Bessel | vs scipy | 3e-16 |
```
```


```
```
| Mie `a_n,b_n` | energy conservation `Qext=Qsca` (lossless) | exact |
| scattering chain | original 
```
```
`T11=b_n`, `T33=−a_n` | confirmed |

```
```
| normalized 
```
```
`P̄, π, τ` | vs scipy, stable at `n=140` | ~1e-13 |

```
```
| plane-wave VSH | reconstructs `x̂ e^{ikz}` | 3e-12 |
| **full general-`m` BSC chain** | vs direct angular spectrum (Eq.10) | **4e-15** |
| normalized (overflow-free) reformulation | vs direct | 
```
```
**3.8e-15** |

```
```
| FFT φ-integration convention | vs brute sum | 8e-15 |

`test_planewave_mie.m`
```
```
 re-checks (A) the field vs the direct angular-spectrum

```
```
integral and (B) Mie energy conservation, inside MATLAB.
```
```


```
```

## Known caveats / to verify when you run it

- **Internal field (`r < a`)** uses Bohren–Huffman `c_n, d_n`, validated by
tangential-field continuity at 
```
```
`r = a`: **per-mode continuity is exact to

```
```
machine precision for all significant modes** (
```
```
`n ≲ x`); only the evanescent

```
```
high-`n` tail shows floating-point noise *exactly at* `r = a` (huge `h_n(x)` ×
tiny `j_n(mx)`), which is harmless away from the surface.
-
```
```
 Accuracy on **curved** source surfaces is the method's inherent

```
```
`O(λ/R)`
```
```
 (planar = exact; radius of curvature ≳ 2λ recommended), unchanged

```
```
from the paper.
```
```


```
```
-
```
```
 For exact bit-for-bit reproduction of the original intermediate numbers you'd

```
```
need the repo's 
```
```
`shiftGB/PIETAOAll/fact`; this rewrite instead matches the

```
```
*physics*
```
```
 (validated above), which is the stronger guarantee.

```
```

## Interface
```
```


```
```

```matlab
out = mieFromSurfaceField(surf, opt)
% surf: .O[Ns x 3] points (rel. sphere centre), .e1/.e3[Ns x 3] local basis,
```
```


```
```
% .E0[Ns x 1] amplitude along e1, .dA[Ns x 1 | scalar] area element
% opt : .k .N .a .mr (+ optional .Reval[P x 3], .nTheta, .nPhi, .verbose,
% .useParallel -- parfor Stage-5 field eval, default false)
```
```


```
```
% out : BSCs (Ate/Ato/Bte/Bto incident, F../G.. scattered, C../D.. internal),
% Mie coeffs an/bn/cn/dn, and (if Reval given) Einc/Escat/Eint/Etot.

```
