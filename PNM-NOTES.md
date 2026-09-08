# uPIMulator — build & run notes

Verified 2026-09-08 on Ubuntu 22.04, kernel 5.15, 80 cores (Go version).
Status: **build OK / run OK** — but only after rebuilding the UPMEM compiler from
source, because the official SDK download is permanently gone.

## THE BLOCKER: UPMEM is out of business
`docker/Dockerfile` does
`wget sdk-releases.upmem.com/2023.2.0/ubuntu_22.04/upmem-2023.2.0-Linux-x86_64.tar.gz`.
The **entire `upmem.com` domain no longer resolves** (`sdk-releases.upmem.com`,
`sdk.upmem.com`, `upmem.com` — all NXDOMAIN). `bongjoonhyun/upimulator` is not on
Docker Hub either. So `docker build` fails and the simulator panics at
`compiler.go:44` before it does anything.

### Workaround that works: build the DPU clang from GitHub
The `upmem` **GitHub org is still alive**, including `upmem/llvm-project`, and the
simulator only needs the compiler *front end* — it invokes
`dpu-upmem-dpurte-clang ... -S` to get assembly and then assembles/links with its
own Go assembler and linker. All DPU headers (stdlib + syslib) are already vendored
in `golang/uPIMulator/sdk/`, so the runtime half of the SDK is not needed.

```bash
mkdir -p .toolchains && cd .toolchains

# Go >= 1.21.5 (system Go 1.18 is too old)
wget https://go.dev/dl/go1.22.5.linux-amd64.tar.gz -O go.tgz && tar xzf go.tgz && rm go.tgz

# UPMEM's LLVM 12 fork, branch matching SDK 2023.2
git clone --depth 1 -b rel_2023.2 https://github.com/upmem/llvm-project.git upmem-llvm
cmake -S upmem-llvm/llvm -B upmem-llvm-build -G Ninja \
  -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_PROJECTS="clang" \
  -DLLVM_TARGETS_TO_BUILD="X86" -DLLVM_EXPERIMENTAL_TARGETS_TO_BUILD="DPU" \
  -DLLVM_ENABLE_ASSERTIONS=OFF -DLLVM_INCLUDE_TESTS=OFF \
  -DLLVM_INCLUDE_BENCHMARKS=OFF -DLLVM_INCLUDE_EXAMPLES=OFF \
  -DCMAKE_INSTALL_PREFIX=$PWD/upmem-clang
# confirm cmake prints "-- Targeting DPU"
ninja -C upmem-llvm-build -j40 clang
ninja -C upmem-llvm-build -j40 install-clang install-clang-resource-headers
```
~25 min on 40 cores; the install is ~116 MB.

Recreate the SDK's two entry points. The driver looks for headers at
`<clang dir>/../share/upmem/include/{stdlib,syslib}` (see
`clang/lib/Driver/ToolChains/DPURTE.h`), so point that at the vendored headers:
```bash
cat > upmem-clang/bin/dpu-upmem-dpurte-clang <<'WRAP'
#!/bin/bash
DPU_CLANG_DIR="$(dirname "$(readlink -f "$0")")"
"${DPU_CLANG_DIR}/clang" --target=dpu-upmem-dpurte "$@"
WRAP
chmod +x upmem-clang/bin/dpu-upmem-dpurte-clang

mkdir -p upmem-clang/share/upmem/include
cp -r ../golang/uPIMulator/sdk/stdlib upmem-clang/share/upmem/include/stdlib
cp -r ../golang/uPIMulator/sdk/syslib upmem-clang/share/upmem/include/syslib
cp -r ../golang/uPIMulator/sdk/misc   upmem-clang/share/upmem/include/misc
cd ..
```

The Go binary hardcodes the container path
`/root/upmem-2023.2.0-Linux-x86_64/bin/dpu-upmem-dpurte-clang`, so drop the built
toolchain into the Docker build context and replace the dead `wget` with a `COPY`:
```bash
cp -r .toolchains/upmem-clang golang/uPIMulator/docker/upmem-2023.2.0-Linux-x86_64
# edit golang/uPIMulator/docker/Dockerfile: replace the wget/tar/echo SDK block with
#   COPY upmem-2023.2.0-Linux-x86_64 /root/upmem-2023.2.0-Linux-x86_64
docker build -t bongjoonhyun/upimulator golang/uPIMulator/docker
```
(The simulator re-runs `docker build` itself on every invocation; it is a no-op once
the layers are cached.)

## Build the simulator
```bash
cd golang/uPIMulator
export GOROOT=$PWD/../../.toolchains/go GOPATH=$PWD/../../.toolchains/gopath
export PATH=$GOROOT/bin:$PATH
python3 script/build.py         # -> build/uPIMulator
```

## Run
`--root_dirpath` and `--bin_dirpath` must be **absolute**, and `bin_dirpath` must
exist and be empty:
```bash
cd golang/uPIMulator
rm -rf bin && mkdir bin
./build/uPIMulator --root_dirpath $PWD --bin_dirpath $PWD/bin --benchmark VA \
  --num_channels 1 --num_ranks_per_channel 1 --num_dpus_per_rank 1 \
  --num_tasklets 16 --data_prep_params 1024
```
Cycle-level stats are written to `bin/log.txt`, e.g.
`Logic[0_0_0]_logic_cycle: 32588`, `Logic[0_0_0]_num_instructions: 13400`,
`MemoryController[0_0_0]_memory_cycle: 195528`, plus row-buffer counters.

Verified benchmarks: VA, GEMV, RED, TS, SEL (13 PrIM kernels ship in `benchmark/`;
BFS, SpMV and NW are excluded upstream). Per-figure `data_prep_params` values are in
`golang/README.md`.

## Not tested
`golang_vm/` has the same dead-SDK Dockerfile and should work with the same fix.
`python_cpp/` additionally wants SDK 2021.3.0 (also gone) plus
`upmem/llvm-project` — same approach, different branch.
