# AWS F2 CVA6 (this fork)

This fork of llama.cpp adds two CVA6-specific knobs used by
[sle-benchmarks `tests/llama`](https://github.com/SilverLining-EDA/sle-benchmarks/blob/main/tests/llama/README.md)
to produce a static `llama.bin` for the F2 `cl_cva6_benchmarks` loader (BAR4 at `0x80000000`, UART report).

Do not use stock ggml-org llama.cpp for that flow; the patches below are required.

## Why the patches exist

This CVA6 HBM port **never completes AMO/LR/SC**. Building with `-march=rv64gc` (A + C) hangs in libc/ggml atomics.

xPack `riscv-none-elf` libstdc++ is **single-thread**. `<future>` / `std::async` / `std::call_once` do not link.

## CMake: `CVA6_MARCH`

[`ggml/src/ggml-cpu/CMakeLists.txt`](../ggml/src/ggml-cpu/CMakeLists.txt) uses `CVA6_MARCH` instead of `rv64gc` when the variable is set.

sle-benchmarks passes:

```
-DCVA6_MARCH=rv64imfd_zicsr
```

(`lp64d`, no `A`, no `C`). GCC then selects the `rv64ifd_zicsr/lp64d` multilib. **Do not use `rv64gc`.**

## C++: `CVA6_BAREMETAL`

Define `-DCVA6_BAREMETAL` on compile of llama.cpp. [`src/llama-model-loader.cpp`](../src/llama-model-loader.cpp) then:

- does not include `<future>`
- validates tensors synchronously (no `std::async`)
- replaces `std::call_once` with a static `bool`

## Host-side build and FPGA run

All remaining steps (xPack GCC, portable CMake, GGUF embed, `make`, `./run.sh`, AGFI, UART) live in the benchmark repo:

https://github.com/SilverLining-EDA/sle-benchmarks/blob/main/tests/llama/README.md

Typical layout on the F2 instance:

| Tree | Path |
|------|------|
| This fork | `/projects/prj1/sle-wajahat/llama.cpp` |
| Benchmarks + `tests/llama` | `/projects/prj1/sle-wajahat/sle-benchmarks` |
| xPack GCC 15.2 | `/projects/prj1/sle-wajahat/tools/xpack-riscv-none-elf-gcc-15.2.0-1` |
| CMake | `/projects/prj1/sle-wajahat/tools/cmake-3.30.5-linux-x86_64/bin/cmake` |

```bash
cd /projects/prj1/sle-wajahat/sle-benchmarks/tests/llama
# place a tiny GGUF at model.gguf, then:
make
export AWS_FPGA_REPO_DIR=/projects/prj1/sle-wajahat/aws-fpga
SKIP_AGFI_LOAD=1 ./run.sh    # if AGFI agfi-0248c1f84010b03e9 is already loaded
```
