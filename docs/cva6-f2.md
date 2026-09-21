# AWS F2 CVA6 (this fork)

This fork adds `CVA6_MARCH` and `CVA6_BAREMETAL` so [NeuQore/benchmarks `tests/llama`](https://github.com/NeuQore/benchmarks/blob/cva6/tests/llama/README.md) can link a static `llama.bin` for the F2 [`cl_cva6_llama`](https://github.com/NeuQore/aws-fpga/tree/cva6/hdk/cl/examples/cl_cva6_llama) interactive loader (BAR4 at `0x80000000`, UART TX-only, HBM mailbox prompts).

**Step-by-step (clone, `setup.sh`, GGUF, `make`, FPGA):** see the [AWS F2 CVA6 section in README.md](../README.md#aws-f2-cva6--step-by-step).

Do not use stock ggml-org llama.cpp for that flow.

## Tool dependencies

Install with [`../setup.sh`](../setup.sh) (no sudo). The system package `gcc-riscv64-unknown-elf` **cannot** build this target.

| Tool | Version / path after `./setup.sh` |
|------|-------------------------------------|
| xPack `riscv-none-elf` GCC | 15.2.0-1 — newlib + libstdc++ |
| CMake | 3.30.5 portable tarball |
| numpy venv | optional, dummy GGUF only |

Then `source $TOOLS_DIR/cva6-env.sh` (`XPACK_ROOT`, `CMAKE`).

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

[`src/llama-vocab.cpp`](../src/llama-vocab.cpp) `byte_to_token` uses `unordered_map::find` instead of `at()` so a missing `<0x0A>` / `<0xXX>` token does not throw (`std::out_of_range` unwind hangs on this libstdc++).
