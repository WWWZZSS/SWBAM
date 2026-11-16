# SWBAM

A high-performance BAM/SAM processing library and distributed applications for the Sunway architecture.
Companion code for **SWBAM: Fast and Scalable BAM/SAM File Manipulation on the Sunway Architecture**.

## Requirements

- Sunway Linux, `swgcc`/`swg++` with C++11 support, MPI, and athread. x86 builds are not supported.
- CMake >= 3.13, Make, Bash, and zlib development files.
- Bundled HTSlib 1.20, libdeflate 1.20, and CLI11 2.5.0. Use the supplied sources to retain the Sunway adaptations.

HTSlib provides header handling and standard alignment data structures; SWBAM manages bulk I/O, batch execution, and adapted CPE compression and parsing kernels.

## Installation

Build on a host with the Sunway toolchain:

```bash
git clone https://github.com/WWWZZSS/SWBAM.git
cd SWBAM
bash scripts/build_dependencies.sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j8
```

Executables are generated in `build/bin/`. The default MPI path is `/usr/sw/mpi/mpi_20220608_SEA`; override it with `-DSUNWAY_MPI_ROOT=/path/to/mpi` if needed.

## Usage

Launch through the site scheduler. This example uses one node with six MPI ranks, each using one MPE and 64 CPEs:

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

Adjust the queue and resources for your site. `SWBAM_NODE_MEMORY` is a node memory budget, not an allocation request; `--work-memory` sets the per-rank budget. Input and output paths must be accessible to all ranks.

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

`view` selects the conversion from the input format and output extension; its options differ from `samtools view`. BAM output uses compression level 1. For `sort` and `collate`, `-m` controls the per-rank algorithm workspace and the choice of in-memory or external-memory execution. For `fixmate`, `-m` enables mate-score tags.

The complete `collate -> fixmate -> sort -> markdup` workflow is also available as one command:

```bash
run "$APP" dedup-workflow -i input.bam -o marked.bam \
  -m 4G -n 66 -l 300 -T "$TMPDIR/dedup"
```

Use a unique `-T` prefix for each workflow; intermediate BAM files are removed on success unless `--keep-intermediates` is set. See `run "$APP" COMMAND --help` for options.

## SDK

The SDK is in `swbam-sdk/`; applications are in `swbam-app/`. The two SDK examples demonstrate [record counting](swbam-sdk/examples/sdk_optimized_count.cpp) and [BAM reading and writing](swbam-sdk/examples/sdk_optimized_bam2bam.cpp):

```bash
run "$PWD/build/bin/swbam-sdk-optimized-count" input.bam
run "$PWD/build/bin/swbam-sdk-optimized-bam2bam" input.bam -o rewritten.bam
```

Install the SDK and build the examples as an external project:

```bash
cmake --install build --prefix "$PWD/install"
cmake -S swbam-sdk/examples -B build-sdk-examples \
  -DCMAKE_TOOLCHAIN_FILE="$PWD/install/lib/cmake/SWBAM/SunwayToolchain.cmake" \
  -DCMAKE_PREFIX_PATH="$PWD/install"
cmake --build build-sdk-examples -j8
```

External projects use `find_package(SWBAM CONFIG REQUIRED)` and link to `swbam::swbam`, which includes the three host libraries and CPE objects. See the [example CMake configuration](swbam-sdk/examples/CMakeLists.txt).

## Tests

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DSWBAM_BUILD_TESTS=ON
cmake --build build -j8
mkdir -p tmp
bash tests/run_smoke.sh build/bin "$PWD/tmp/smoke-01" \
  bsub -q q_share -I -b -N 1 -np 6 -cgsp 64 \
  -share_size 14000 -cache_size 32 -x SWBAM_NODE_MEMORY=72G
```

Use a new output directory and a synchronous launcher. This 20,000-record correctness regression covers the applications and SDK examples, reporting `SWBAM_SMOKE PASS records=20000 duplicates=10000` on success. It is not a full reproduction of the paper's experiments.
