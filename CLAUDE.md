# CalculiX — Claude Session Notes

Living context file for Claude Code sessions on this repo. Read this before starting work — it captures what's already been investigated, what works, what doesn't, and what the user wants next.

**Last updated**: 2026-05-27
**Branch**: master (clean tracking origin/master)
**Owner**: basem (b.rajjoub@gmail.com)

---

## 1. What this project is

CalculiX is a 3D finite-element analysis (FEA) solver. Mostly Fortran, some C. Maintained primarily by Guido Dhondt since 1998 (390 of ~410 commits). GPL-2.

- **Total source**: ~337K LOC across 1182 files in `src/`
- **Fortran**: 987 files, ~251K LOC
- **C**: 186 files, ~80K LOC
- **Headers**: 9 files, ~6.4K LOC
- **Test suite**: 638 `.inp` cases + 931 `.ref` reference outputs in `test/`

## 2. Architecture overview

### Entry points
- `src/CalculiX.c:34` — `int main(int argc,char *argv[])`. Reads `-i jobname`, calls `openfile`, loops over steps via `CalculiXstep()`.
- `src/CalculiXstep.c` — dispatches by `nmethod` (1–12: static, dynamic, frequency, heat, electromagnetics, …).

### Parallel model (important — agents will hallucinate about this)
- **pthreads, NOT OpenMP**. 44 C files call `pthread_create`. 19 `*main*.c` files are pthread fan-out wrappers.
- **0 `!$OMP` directives in Fortran**. (An exploration agent claimed 74,502 — false, verified by grep.)
- **16 `#pragma omp` lines only in `src/pastix.c`** — the one exception.
- Hub pattern: `mafillsmmain.c` reads `CCX_NPROC_STIFFNESS` then falls back to `OMP_NUM_THREADS` as a thread-count source (OpenMP is not actually invoked — just the env var is reused). Spawns `num_cpus` pthreads, joins, accumulates per-thread buffers.
- Same pattern repeats in: rhsmain, results, resultsem, mafilldmss, mafillsmas, mulmatvec_asym, opmain, biosav, compfluidfem, electromagnetics, dudsmain, filterforwardmain, filterbackwardmain, interpolatestatemain, interpoltetmain, mafillsmmain_se, packagingmain, randomfieldmain, thicknessmain, transitionmain.

### Build flavors (upstream)
| Makefile | Target | Flags | Notes |
|---|---|---|---|
| `src/Makefile` | `CalculiX` | `-Wall -g` (no -O!) | Default. Effectively debug build. `-fopenmp` appears only on link, not compile. |
| `src/Makefile_MT` | `CalculiX_MT` | `-Wall -O2 -fopenmp -cpp` | Production multi-thread. Links spoolesMT.a. |
| `src/Makefile_MFront` | `libCalculiX_helfer.so` | `-Wall -O3 -fopenmp -fPIC` | Shared lib + MFront material framework. |
| `src/Makefile_i8` | `CalculiX_i8` | `-O2 -fopenmp -fpic -DINTSIZE64 -DPASTIX -DPASTIX_GPU` | 64-bit ints + PaStiX + CUDA GPU. |

### External deps
- **SPOOLES.2.2** (1999, archaic C) — sparse direct solver. Default. Not in any distro package.
- **LAPACK 3.12.x** — Fedora `lapack-static` provides `/usr/lib64/liblapack.a`.
- **BLAS / refblas** — Fedora `blas` ships `/usr/lib64/libblas.so` (shared only — no static). System routes via flexiblas.
- **ARPACK** — Fedora `arpack-devel` ships `/usr/lib64/libarpack.so`. Static needs `arpack-static`.
- **PaStiX + CUDA** — optional, only in Makefile_i8.

## 3. What we did this session

### Investigation phase
- Mapped build system, parallel architecture, hot paths.
- Spawned 3 Explore agents in parallel; **two hallucinated badly** (claimed 74,502 OMP directives, claimed wrong file paths). Always verify agent claims with grep before trusting.
- Verified: default Makefile genuinely has no -O flag; OMP claim is fake; pthreads is the real model.

### Plan phase (canceled by user)
- Designed a 6-phase optimization roadmap with A/B harness, unit_tests folder, Makefile_OPT variant.
- User wanted critical reality check — found that:
  - "Phase 1 = 2-4× speedup" was misleading (only true vs broken debug build; -O2→-O3 is realistically 10–25%).
  - OMP-inside-pthread = nested parallelism, no-op by default.
  - BLAS swap on 60×60 blocks = function-call overhead > work.
  - Atomic-add race fix for `extrapolate.f` = likely net slowdown.
  - Honest total cumulative gain: 1.3–1.8×, not the 4–8× agents implied.

### Build phase (succeeded)
Successfully built CalculiX from source on Fedora 44 + GCC 16 + Ryzen 7 4700U (8 cores, 15GB RAM):

```bash
# 1. Install Fedora packages
sudo dnf install lapack-static arpack-devel
# (also pulls blas, lapack, openblas64, flexiblas-*, ~56MB)

# 2. Build SPOOLES from netlib tarball
mkdir -p /home/basem/Desktop/github/SPOOLES.2.2
cd /home/basem/Desktop/github/SPOOLES.2.2
wget http://www.netlib.org/linalg/spooles/spooles.2.2.tgz
tar xzf spooles.2.2.tgz
sed -i 's|CC = /usr/lang-4.0/bin/cc|CC = cc|' Make.inc
# Patch CFLAGS for GCC 16 strictness:
sed -i 's|^  OPTLEVEL = -O$|  OPTLEVEL = -O -fcommon -Wno-error=implicit-int -Wno-error=int-conversion -Wno-error=incompatible-pointer-types -Wno-error=implicit-function-declaration|' Make.inc
make lib -j8                                    # ~16s, produces spooles.a (2.4MB)

# 3. Build CalculiX default flavor
cd /home/basem/Desktop/github/CalculiX/src
make -f Makefile_local -j8                      # ~1m 9s, produces src/CalculiX
```

#### Timings (measured)
| Operation | Time |
|---|---|
| Per-file compile (Fortran, -O0, avg) | 0.15s |
| Per-file compile (Fortran, -O3 -march=native, big files like e_c3d.f) | **6.16s** (~19× slower than -O0) |
| Per-file compile (C, -O0, avg) | 0.15s |
| Full `make -j8` (-O0) | **~1m 9s** |
| Full `make -j8` (-O3 -march=native) — estimated | 3–6 min |
| Incremental rebuild after 1 .c file | 10–30s |
| SPOOLES build | 16s |
| beam8p smoke test (1200 DOF) | 0.07s solve |
| cubef2f2 mid-size test | 9.2s solve |

### Shared library experiment — succeeded
Built `libcalculix.so` proof-of-concept:

```bash
cd /home/basem/Desktop/github/SPOOLES.2.2
# Rebuild SPOOLES with -fPIC (required for .so linking)
sed -i 's|^  OPTLEVEL = -O -fcommon|  OPTLEVEL = -O -fPIC -fcommon|' Make.inc
make clean && make lib -j8                      # ~20s

cd /home/basem/Desktop/github/CalculiX/src
# Make .so source with main renamed
sed 's/^int main(int argc,char \*argv\[\])/int calculix_main(int argc,char *argv[])/' CalculiX.c > CalculiX_so.c
make -f Makefile_so -j8                         # ~34s, produces libcalculix.so (13MB)
```

Tested via `dlopen` + `dlsym("calculix_main")` from `/tmp/ccxtest/driver.c`. Solves `beam8p` in 0.11s — identical output to the binary.

**Caveats — DO NOT mislead the user**:
- **239 `exit()` calls in C + 717 `STOP` calls in Fortran** = any error path kills the host process.
- Hundreds of file-scope `static` globals = **not re-entrant**. Calling `calculix_main()` twice in the same process = undefined behavior.
- Hardcoded file I/O — reads `<jobname>.inp`, writes `<jobname>.dat`/`.frd`/`.sta` to CWD. No in-memory API.
- For real integration: fork a child process per solve, dlopen there. Or accept subprocess pattern.
- **GPL contagion**: linking libcalculix.so into another binary forces GPL on the consumer.

## 4. User's broader project context

The user is building **feview**: a Rust + egui + wgpu desktop GUI for FEM at `/home/basem/Desktop/github/feview`.

- Already has its own linear static solver (`src/solver/mod.rs` — rayon parallel assembly + PCG with K-matrix caching).
- Calls CalculiX as subprocess via `src/solver/calculix.rs` — `find_ccx()` checks `~/.pixi/bin/ccx` first (pixi-installed CalculiX 2.22), then `PATH`. Writes `.inp` to temp dir, spawns ccx, parses `.dat`.
- User's stated goal: optional **immediate-mode** for small/educational models (≤ 2k DOF). Big models stay subprocess.

### Plan agreed in conversation (NOT yet implemented)
- Feview's existing Rust PCG solver handles immediate-mode (no CalculiX changes needed for this).
- Two-mode toggle in UI: immediate-mode ON → Rust PCG with debounce; OFF → ccx subprocess "Run" button.
- Auto-disable immediate mode when DOF > threshold.
- Effort estimate: ~2.5 days in feview repo. Zero changes to CalculiX.
- **CalculiX integration NOT chosen for immediate-mode path** — fixed 100-250ms subprocess overhead too slow, but Rust solver already covers small linear static.

### Future integration paths (ranked, none started)
1. **JSON progress stream** (1 day) — patch CalculiX to emit per-Newton-iteration JSON on stdout. Feview parses lines → live residual plot. Best UX bang/buck.
2. **CalculiX daemon mode** (3–5 days) — `-server` flag, reads SOLVE commands on stdin, stays alive. Saves ~80ms fork+exec per solve.
3. **libcalculix.so deep integration** (2–4 weeks) — already proven viable today; needs re-entrancy refactor + GPL decision.
4. **Immediate-mode-with-progress** (2–3 days on top of 1) — stream intermediate displacements, watch deformation grow.

## 5. Other features evaluated this session

User asked about CalculiX extensibility and missing features (forget performance):

### Extension points (ranked by ease)
| Feature | Difficulty | Where |
|---|---|---|
| New `*COMMAND` keyword | Easy (hours) | `src/calinput.f` (1863 lines, 134 `elseif` branches) |
| New user material | Easy | Existing umat hooks: `src/umat.f`, `src/umat_main.f`, 18 built-in `umat_*.f` variants; MFront support |
| Other user hooks | Easy | `src/uamplitude.f`, `src/uboun.f`, `src/umatht.f` |
| New element type | Moderate (days–weeks) | Touches ~15–25 files: shape*.f, e_c3d.f (2366 lines of lakon dispatch), mafillsm*.f, extrapolate.f, output writers |
| New PDE / custom equations | Hard | No general-equation layer. Must clone the mafill* + assembly + solver-wiring pattern (see CFD: `compfluidfem.c` + `mafillv*/p*/k*/t*.f`) |

### Missing features ranked by impact ÷ effort
| Feature | Effort | Why valuable |
|---|---|---|
| **CI-friendly test runner** | 1–2 days | 638 cases unused for regression gating. No exit code in `test/compare`. |
| **VTK/VTU native output** | 5 days (or 0 if using existing converter) | FRD is niche. ParaView needs converter today. |
| **Python bindings (libccx + pybind11)** | 1–2 weeks | Killer for scripted parameter sweeps + research adoption. |
| **JSON progress / structured logging** | 2–3 days | Enables web dashboards, queue systems, GUI progress bars. |
| **JSON material catalog `*MATERIAL, JSON=mat.json`** | 3 days | Reusable, version-controllable, diffable. Pure parser work. |
| **Better restart robustness** | 1 week | Long runs need this. |
| **Topology-sensitivity export** | 1 week | `_se` files already half-built. |
| **libccx as embeddable C API** | 2 weeks | Cloud-native. Big lift. |
| **Adaptive mesh refinement (h-adaptive)** | 4–8 weeks | `.rfn` infra partial; full feature = publishable. |
| **Contact diagnostics output** | 3 days | Data exists, just needs export. |

### Existing FRD→VTK converters (do not reinvent!)
- **[ccx2paraview](https://github.com/calculix/ccx2paraview)** — official, GPL-3, v3.2.0 (Dec 2024), 125★. Python. ASCII-FRD only. One file per timestep.
- **[wr1/frd2vtu](https://github.com/wr1/frd2vtu)** — MIT, v0.2.0 (Apr 2026). Binary FRD support (142/144 element types).
- **[precice/calculix-adapter frdToVTKConverter](https://github.com/precice/calculix-adapter/tree/master/tools/frdConverter)** — C#.
- Native ParaView FRD reader — requested in 2021, not done.

User installed ccx2paraview via pixi already. Best path is auto-invoke at solve end, not native rewrite.

## 6. Files added by this session (committed)

- `src/CalculiX_so.c` — copy of `CalculiX.c` with `main` → `calculix_main` (66KB). Built by `Makefile_so`.
- `src/Makefile_local` — local build recipe; uses Fedora system libs + locally built SPOOLES. Produces `src/CalculiX`.
- `src/Makefile_so` — shared-lib build recipe. Produces `src/libcalculix.so`.
- `.gitignore` updated to also exclude `src/CalculiX` (binary).
- This file (`CLAUDE.md`).

**Not committed** (gitignored): all `.o`, `.a`, `.so`, the `CalculiX` binary itself.

**Out-of-repo deps**: `/home/basem/Desktop/github/SPOOLES.2.2/` (built; not in git).

## 7. Pixi-installed CalculiX (separate from this repo)

The user also has `~/.pixi/bin/ccx` (CalculiX 2.22 from conda-forge). Feview's `find_ccx()` checks this path first. It's a separate working CalculiX install — useful as a sanity reference.

## 8. Open questions / decisions pending

These are unresolved threads that the next session may want to pick up:

1. **Compiler choice**: stick with gfortran (current), or also support ifx + Intel MKL? Affects all optimization phases.
2. **MKL license posture**: are we OK depending on Intel MKL for PARDISO?
3. **`-march=native` portability**: binaries built on Ryzen 4700U use AVX2; won't run on older AVX-only CPUs. Issue if distributing.
4. **Test set scope**: full suite (638 cases) takes hours. Need a 5–8 case subset for fast iteration (potential picks: beamnlmpc, contact1, acou1, cubef2f2, crackIIcum, hueeber1, hueeber4_mortar, achtel29).
5. **Tolerance for unit_tests**: `1e-10` relative diff? `1e-12`? `-O3 -ftree-vectorize` reassociates FP so stricter may flake.
6. **GPL implication**: linking libcalculix.so into feview forces feview to GPL-3. Feview's Cargo.toml has no license set yet. Decide before deep integration.

## 9. Conventions / preferences observed

- User wants **terse, critical, honest** responses. Push back on bad estimates from upstream agents. Caveman mode active.
- User prefers **additive changes** over modifying upstream-tracked files (so `Makefile_local`, `Makefile_so`, `CalculiX_so.c` rather than editing the originals).
- User values **A/B testability** — keep old code alongside new so we can swap and time both.
- User values clean repo: revert `date.pl` timestamp bumps before commit (auto-regenerated next build, just noise).
- Upstream is conservative (Guido Dhondt commits 390/410 = 95%). Patches unlikely to merge → stays as fork. Maintenance burden grows over time.

## 10. Quick reference: rebuild from scratch

If the binaries are gone and you need to rebuild on a fresh checkout:

```bash
# Deps
sudo dnf install lapack-static arpack-devel

# SPOOLES (~16s)
mkdir -p /home/basem/Desktop/github/SPOOLES.2.2
cd /home/basem/Desktop/github/SPOOLES.2.2
wget http://www.netlib.org/linalg/spooles/spooles.2.2.tgz
tar xzf spooles.2.2.tgz
sed -i 's|CC = /usr/lang-4.0/bin/cc|CC = cc|' Make.inc
sed -i 's|^  OPTLEVEL = -O$|  OPTLEVEL = -O -fcommon -Wno-error=implicit-int -Wno-error=int-conversion -Wno-error=incompatible-pointer-types -Wno-error=implicit-function-declaration|' Make.inc
make lib -j8

# CalculiX binary (~1m 9s)
cd /home/basem/Desktop/github/CalculiX/src
make -f Makefile_local -j8

# OR libcalculix.so (~34s + SPOOLES with -fPIC ~20s)
cd /home/basem/Desktop/github/SPOOLES.2.2
sed -i 's|OPTLEVEL = -O -fcommon|OPTLEVEL = -O -fPIC -fcommon|' Make.inc
make clean && make lib -j8
cd /home/basem/Desktop/github/CalculiX/src
make -f Makefile_so -j8
```

## 11. Where to look next

- `src/Makefile_local`, `src/Makefile_so` — our build recipes.
- `src/CalculiX_so.c` — the only Fortran/C source we added; rest is upstream.
- `feview/src/solver/calculix.rs` — feview's existing subprocess integration.
- `feview/CLAUDE.md` — feview's own session notes (separate repo).
