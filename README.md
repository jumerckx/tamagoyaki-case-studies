# Tamagoyaki case studies

Two compilers built on [Tamagoyaki](https://github.com/jumerckx/Tamagoyaki), the
MLIR framework for e-graphs as a persistent compiler abstraction, together with
the evaluations from the paper.

| | |
|---|---|
| `herbie_mlir/` | floating-point accuracy optimisation in the spirit of [Herbie](https://herbie.uwplse.org/), with interval arithmetic from [Rival 3](https://github.com/herbie-fp/rival3) |
| `rover-mlir/`  | datapath optimisation over [CIRCT](https://circt.llvm.org/)'s `comb`/`hw`/`datapath` dialects |
| `tamagoyaki_eval/` | what the two Snakemake pipelines share: timing, provenance, and the rules neither owns alone |

They live here rather than in the Tamagoyaki repository because of what they
drag in — CIRCT, Rust, Rival 3, a pinned Herbie and its whole Racket package
closure — none of which the framework itself needs. Splitting them out lets the
two version independently: this repository pins the Tamagoyaki it was tested
against in `flake.lock`, and bumping it is a deliberate, reviewable step.

## Building

Everything is declared in the Nix flake, Tamagoyaki included — it arrives
prebuilt from the pinned input, along with the exact MLIR it was compiled
against, so nothing here builds a second LLVM.

```shell
nix develop
case-studies-configure build
ninja -C build check-all
```

To work against a local Tamagoyaki checkout instead of the pinned one, override
the input. Its *build* directory exports a CMake config too, so there is no
install step in the inner loop:

```shell
nix develop --override-input tamagoyaki path:../Tamagoyaki
case-studies-configure build \
  -DTamagoyaki_DIR=../Tamagoyaki/build/lib/cmake/tamagoyaki
```

Either case study can be switched off — `-DBUILD_HERBIE_MLIR=OFF`,
`-DBUILD_ROVER_MLIR=OFF` — which is how the two evaluation builds avoid paying
for each other's dependencies.

## Running the evaluations

The pipelines build nothing; they need a compiler and a place to put results.

```shell
make eval          # both, one after the other
make herbie-eval   # just the Herbie-MLIR pipeline    -> herbie-eval-out/
make rover-eval    # just the Rover datapath pipeline -> rover-eval-out/
```

Both are Snakemake workflows: `herbie_mlir/eval/Snakefile` and
`rover-mlir/eval/Snakefile`. `herbie-eval` and `rover-eval` take Snakemake
arguments directly, which is the way to run part of one.

For the packaged artifact — a Docker image that runs both offline — see
[`docs/artifact.md`](docs/artifact.md).

## Provenance

Each run writes a manifest recording `git_rev` (this repository) **and**
`tamagoyaki_rev` (the framework revision under measurement, read from
`flake.lock`). Both are needed to reproduce a number; neither alone is enough.
