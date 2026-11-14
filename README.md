# SWBAM

[Chinese translation](README.zh-CN.md)

Source code accompanying **SWBAM: Fast and Scalable BAM/SAM File Manipulation on the Sunway Architecture**.
SWBAM provides a reusable BAM/SAM processing library and distributed applications for the Sunway heterogeneous many-core architecture. It supports flag statistics, filtering, format conversion, coordinate sorting, QNAME collation, mate-field correction, duplicate marking, and a four-stage deduplication workflow.

## Requirements

- Sunway Linux with `swgcc`, `swg++` (C++11), MPI, and athread; x86 builds are not supported.
- CMake >= 3.13, GNU Make, Bash, and zlib development files.
- Bundled dependencies: adapted HTSlib 1.20, libdeflate 1.20, and CLI11 2.5.0. Use the supplied sources to preserve the Sunway adaptations.

The default MPI installation is `/usr/sw/mpi/mpi_20220608_SEA`; override it with `-DSUNWAY_MPI_ROOT=/path/to/mpi` when configuring SWBAM.

## Build

Run from the repository root on a host with the Sunway toolchain:

```bash
bash scripts/build_dependencies.sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j8
```

Executables are generated in `build/bin/`: `SWBAM`, `swbam-sdk-optimized-count`, and `swbam-sdk-optimized-bam2bam`.
Dependencies are built in `build/dependencies/`. To use another directory, pass it to `build_dependencies.sh` and set the same path through `-DSWBAM_DEPENDENCY_BUILD_DIR=...`.

## Run

Launch executables through the site scheduler. The following example uses one node with six MPI ranks, each running one MPE and 64 CPEs. Adjust the queue and resource settings for your site. Input and final output paths must be accessible to all ranks.

```bash
mkdir -p tmp
export TMPDIR="$PWD/tmp"
APP="$PWD/build/bin/SWBAM"

run() {
  bsub -q q_share -I -b -N 1 -np 6 -cgsp 64 \
    -share_size 14000 -cache_size 32 \
    -x SWBAM_NODE_MEMORY=72G "$@" --work-memory 8G </dev/null
}
```

`SWBAM_NODE_MEMORY` specifies the usable memory budget per node, not an allocation request. The example conservatively budgets 12 GiB per rank; never exceed the memory actually assigned to the job. `--work-memory` sets the per-rank working budget.

### Applications

```bash
run "$APP" flagstat -i input.bam
run "$APP" run_all -i input.bam -o rewritten.bam
run "$APP" run_all -i input.bam -o filtered.bam --min-mapq 30
run "$APP" run_all -i input.bam -o output.sam
run "$APP" run_all -i input.sam -o output.bam
run "$APP" collate -i input.bam -o names.bam -m 4G -n 66 -T "$TMPDIR/collate"
run "$APP" fixmate -i names.bam -o mates.bam -m
run "$APP" sort -i mates.bam -o sorted.bam -m 4G -T "$TMPDIR/sort"
run "$APP" markdup -i sorted.bam -o marked.bam -m 4G -l 300
```

`run_all` selects the conversion from the input format and output extension. BAM output uses compression level 1. `sort` and `collate` select in-memory or external-memory execution according to the per-rank algorithm budget `-m`; for `fixmate`, `-m` instead enables mate-score tags. `markdup` always uses the single-pass path.

To run the complete `collate -> fixmate -> sort -> markdup` workflow:

```bash
run "$APP" dedup-workflow -i input.bam -o marked.bam \
  -m 4G -n 66 -l 300 -T "$TMPDIR/dedup"
```

The workflow uses intermediate BAM files and removes them after successful completion unless `--keep-intermediates` is supplied. Use a unique `-T` prefix for each workflow. Subcommand options are available with `run "$APP" COMMAND --help`.

### SDK Examples

```bash
run "$PWD/build/bin/swbam-sdk-optimized-count" input.bam
run "$PWD/build/bin/swbam-sdk-optimized-bam2bam" input.bam -o rewritten.bam
```

These examples perform read-only record counting and an unfiltered BAM read-write round trip. The SDK exposes three host libraries (`swbam_io`, `swbam_mpi_runtime`, and `swbam_cpe_runtime`) and CPE objects through the CMake target `swbam::swbam`.

To install the SDK and build the examples against the installed package:

```bash
cmake --install build --prefix "$PWD/install"
cmake -S swbam-sdk/examples -B build-sdk-examples \
  -DCMAKE_TOOLCHAIN_FILE="$PWD/install/lib/cmake/SWBAM/SunwayToolchain.cmake" \
  -DCMAKE_PREFIX_PATH="$PWD/install"
cmake --build build-sdk-examples -j8
```

External CMake projects use `find_package(SWBAM CONFIG REQUIRED)` and link to `swbam::swbam`.

## Tests

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DSWBAM_BUILD_TESTS=ON
cmake --build build -j8
mkdir -p tmp
bash tests/run_smoke.sh build/bin "$PWD/tmp/smoke-01" \
  bsub -q q_share -I -b -N 1 -np 6 -cgsp 64 \
  -share_size 14000 -cache_size 32 -x SWBAM_NODE_MEMORY=72G
```

Use a new output directory and a launcher that waits for completion and returns the job's exit status. The smoke test checks conversion, filtering, in-memory/external sort and collate, fixmate tags, exact duplicate membership, the workflow, and SDK examples using 20,000 synthetic records. Success is reported as `SWBAM_SMOKE PASS records=20000 duplicates=10000`. This is a small correctness regression, not a full reproduction of the paper's experiments.

## I/O and Input Constraints

- **Each BAM record must fit entirely within one BGZF block.** Arbitrary BAM files are not guaranteed to satisfy this constraint. CRAM is not supported.
- `fixmate` requires QNAME-collated input. `markdup` requires coordinate-sorted input with mate tags prepared by `fixmate -m`; `-l` must cover the displacement between alignment-start and unclipped 5' coordinates.
- Input supports `--io-backend posix|mpiio` (default: `posix`); final output uses MPI-IO. This release performs real file I/O without whole-file RAM simulation or deferred dumps.
- Rank-local output staging defaults to temporary files. `--rank-body-backend memory|spool|auto` selects its policy; `auto` spills at `--rank-body-memory-limit`, whereas `memory` retains the complete local body. Use `--rank-body-temp-dir` for these files and `-T` for sort/collate runs or workflow intermediates. Temporary directories must exist and have sufficient space.
- Algorithm workspaces, the global BGZF index, and SAM input buffers still consume memory. SAM-to-BAM currently buffers each rank's assigned text range; not all memory use is independent of input size.
- `[swbam-processing]` includes actual I/O, computation, and communication but excludes MPI initialization/finalization; `[swbam-e2e]` includes both. Use the maximum rank time. SDK `core` timings cover only the local pipeline, not the complete operation.

## Source Layout and Notices

`swbam-sdk/` contains the library, CPE kernels, and examples; `swbam-app/` contains applications; `third_party/` contains dependency sources; `scripts/` and `tests/` provide build and regression tools.

Third-party licenses and attribution are retained; see [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md). A project-wide license for SWBAM-authored code has not yet been assigned.
