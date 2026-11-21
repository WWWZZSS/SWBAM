# SWBAM

A fast and scalable library and application suite for BAM/SAM file manipulation on the Sunway heterogeneous many-core architecture.

Companion code for **SWBAM: Fast and Scalable BAM/SAM File Manipulation on the Sunway Architecture**.

## Requirements

- Sunway Linux, `swgcc`/`swg++` with C++11 support, MPI, and athread. x86 builds are not supported.
- CMake 3.13 or later, Make, Bash, and zlib development files.
- Bundled HTSlib 1.20, libdeflate 1.20, and CLI11 2.5.0. Use the bundled source code to retain the Sunway-specific adaptations.

HTSlib provides header handling and standard alignment data structures, while SWBAM implements bulk I/O, batch-based execution, and optimized CPE kernels for compression, decompression, and record processing.

## Installation

Build SWBAM on a Sunway system with the Sunway toolchain configured:

```bash
git clone https://github.com/WWWZZSS/SWBAM.git
cd SWBAM
bash scripts/build_dependencies.sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j8
```

The executables are generated in `build/bin/`. The default MPI installation path is `/usr/sw/mpi/mpi_20220608_SEA`. Override it with `-DSUNWAY_MPI_ROOT=/path/to/mpi` if needed.

## Usage

Launch SWBAM through the site scheduler. The following example uses one node with six MPI ranks, each utilizing one MPE and 64 CPEs:

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

Adjust the queue and resource settings according to your system configuration. `SWBAM_NODE_MEMORY` specifies the node-level memory budget rather than a scheduler memory allocation, while `--work-memory` sets the per-rank workspace budget. Input and output paths must be accessible to all MPI ranks.

The following examples demonstrate the available BAM/SAM processing operations:

```bash
run "$APP" flagstat -i input.bam
run "$APP" view -i input.bam -o filtered.bam --min-mapq 30
run "$APP" view -i input.bam -o output.sam
run "$APP" view -i input.sam -o output.bam
run "$APP" collate -i input.bam -o names.bam -m 4G -n 66 -T "$TMPDIR/collate"
run "$APP" fixmate -i names.bam -o mates.bam -m
run "$APP" sort -i mates.bam -o sorted.bam -m 4G -T "$TMPDIR/sort"
run "$APP" markdup -i sorted.bam -o marked.bam -m 4G -l 300
```

The `view` command selects the conversion based on the input format and output filename extension; its options differ from those of `samtools view`. BAM output uses compression level 1. For `sort` and `collate`, `-m` specifies the per-rank algorithm workspace and determines whether in-memory or external-memory processing is used. For `fixmate`, `-m` enables the generation of mate-score tags.

`sort` and the sort stage of `dedup-workflow` use exact global splitters by default.
Each rank receives the same number of records, to within one record. This does not
guarantee equal byte counts or processing times. Use `--sample-splitters` to select
splitters by regular sampling instead.

SWBAM also provides a single command to execute the complete four-stage deduplication workflow:

```bash
run "$APP" dedup-workflow -i input.bam -o marked.bam \
  -m 4G -n 66 -l 300 -T "$TMPDIR/dedup"
```

Use a unique `-T` prefix for each workflow invocation. Intermediate BAM files are removed upon successful completion unless `--keep-intermediates` is specified. Run `run "$APP" COMMAND --help` for the available command-line options.

## SDK

The SDK is provided in `swbam-sdk/`, while the applications are located in `swbam-app/`. Two SDK examples demonstrate [BAM record counting](swbam-sdk/examples/sdk_count.cpp) and [BAM reading and writing](swbam-sdk/examples/sdk_bam2bam.cpp).

```bash
run "$PWD/build/bin/swbam-sdk-count" input.bam
run "$PWD/build/bin/swbam-sdk-bam2bam" input.bam -o rewritten.bam
```

To install the SDK and build the examples as an external project:

```bash
cmake --install build --prefix "$PWD/install"
cmake -S swbam-sdk/examples -B build-sdk-examples \
  -DCMAKE_TOOLCHAIN_FILE="$PWD/install/lib/cmake/SWBAM/SunwayToolchain.cmake" \
  -DCMAKE_PREFIX_PATH="$PWD/install"
cmake --build build-sdk-examples -j8
```

External projects can locate SWBAM using `find_package(SWBAM CONFIG REQUIRED)` and link against `swbam::swbam`, which provides the three host-side libraries and the required CPE objects. See the [example CMake configuration](swbam-sdk/examples/CMakeLists.txt).

## Tests

Build SWBAM with regression tests enabled:

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DSWBAM_BUILD_TESTS=ON
cmake --build build -j8
mkdir -p tmp
bash tests/run_smoke.sh build/bin "$PWD/tmp/smoke-01" \
  bsub -q q_share -I -b -N 1 -np 6 -cgsp 64 \
  -share_size 14000 -cache_size 32 -x SWBAM_NODE_MEMORY=72G
```

Use a fresh output directory and a synchronous job launcher. The 20,000-record correctness regression covers the applications and SDK examples. A successful run reports:

```text
SWBAM_SMOKE PASS records=20000 duplicates=10000
```

This regression test provides a basic correctness check and is not intended to reproduce the full set of experiments reported in the paper.
