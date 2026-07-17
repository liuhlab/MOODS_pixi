# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

MOODS (Motif Occurrence Detection Suite) — C++ algorithms for matching position weight matrices (PWMs) against DNA sequences, exposed to Python via SWIG. This is the `liuhlab/MOODS_pixi` fork, consumed by the `bolero` repo as a git dependency:

```toml
"MOODS-python @ git+https://github.com/liuhlab/MOODS_pixi.git"
```

## Build & install

Packaging is driven by the root `pyproject.toml` (scikit-build-core backend) + `CMakeLists.txt`. A plain install runs SWIG and compiles the extensions automatically — there are no manual pre-build steps:

```bash
pip install .            # from repo root; also how git installs resolve
```

Requirements: a C++ compiler. `swig`, `cmake`, and `ninja` are fetched from PyPI automatically during a build-isolated install (add system `swig` if you disable build isolation).

**In-place dev loop (faster iteration on C++/SWIG):** the legacy scripts still work and build the extensions directly under `python/MOODS/` without a full reinstall. This path needs **system `swig`** on PATH and uses `python/setup.py`:

```bash
cd python && ./build.sh   # runs swig on core/*.i, then setup.py build_ext --inplace
```

## Running / smoke test

There is **no unit-test suite**. Validate changes by scanning the bundled example data with the installed CLI:

```bash
moods-dna.py -m example-data/matrices/*.{pfm,adm} -s example-data/seq/chr1-5k-55k.fa -p 0.0001
```

Note: the `python/scripts/ex-*.py` and `python/paper-test-chr.py` example scripts are **Python 2** (`print` statements, `xrange`, `time.clock()`) and do not run under Python 3 as-is. Only `moods-dna.py` is Python 3-compatible.

## Architecture

**C++ core (`core/`) → SWIG → three Python submodules.** Each `core/*.i` interface declares `%module <name>`, which SWIG turns into `MOODS/<name>.py` + a compiled `_<name>` extension. The generated wrappers are **not committed** — they are produced at build time, so any change to `core/` requires a rebuild before Python sees it.

- **`MOODS.tools`** (`moods_tools.cpp`) — matrix/statistics utilities: `log_odds`, `reverse_complement`, `bg_from_sequence_dna`, and `threshold_from_p` (dynamic-programming conversion of a p-value into a score threshold).
- **`MOODS.parsers`** (`moods_parsers.cpp`) — matrix file readers: `pfm_to_log_odds` (JASPAR `.pfm`, 0-order), `adm_to_log_odds` (high-order `.adm`).
- **`MOODS.scan`** (`moods_scan.cpp` + `scanner.cpp` + `motif_0.cpp` + `motif_h.cpp`) — the scanning engine: the `Scanner` class plus convenience free functions (`scan_dna`, `scan_best_hits_dna`, `naive_scan_dna`).

**Core types (`core/moods.h`):** `score_matrix = vector<vector<double>>` (rows = alphabet symbols, columns = motif positions) is the fundamental object threaded through the entire Python API. `bits_t = uint_fast32_t` backs the bit-parallel scanning.

**Scanning design:** `Scanner` matches *all* motifs simultaneously against a fixed-width lookahead **window** (the `window_size` argument, e.g. `7`): it bit-parallel tests the window across motifs, then verifies candidates into full hits. Motifs are polymorphic behind the abstract `Motif` base (`core/motif.h`) with two implementations — `Motif0` (standard 0-order PWM, `motif_0.cpp`) and `MotifH` (high-order q-gram PWM, `motif_h.cpp`). `Scanner` also supports SNP/indel variant matching (`match_types.h` defines `match`, `variant`, `match_with_variant`).

## Repository layout notes

- `core/` — all C++ sources, headers, and `.i` SWIG interfaces (shared by both build paths).
- `python/MOODS/__init__.py` — empty package marker; the importable `MOODS` package is assembled at build time from this plus the SWIG-generated `.py` and compiled `.so` files.
- `core/misc.i` exists and `python/build.sh` runs SWIG on it, but no `_misc` extension is ever built and nothing imports `MOODS.misc` — it is vestigial. The CMake build omits it.
- `-march=native` is used for optimization, so builds are host-specific by design (this is a source/git-installed package, not a portable wheel).
- Two build configs coexist: root `pyproject.toml` + `CMakeLists.txt` (primary, used by `pip`/git installs) and the legacy `python/setup.py` + `python/build.sh` (in-place dev only).
