# llama.cpp

**This is the SilverLining-EDA fork.** Use it for AWS F2 CVA6 bare-metal (`CVA6_MARCH`, `CVA6_BAREMETAL`). Upstream ggml-org llama.cpp will not build that target.

## AWS F2 CVA6 — step by step

Host FPGA load, GGUF embed, and UART live in [sle-benchmarks `tests/llama`](https://github.com/SilverLining-EDA/sle-benchmarks/blob/main/tests/llama/README.md). Patch notes: [docs/cva6-f2.md](docs/cva6-f2.md).

### Tool dependencies

The Ubuntu package `gcc-riscv64-unknown-elf` has **no newlib and no libstdc++**. Do not use it for this build. `setup.sh` installs portable copies (no sudo):

| Dependency | Why | Installed by `./setup.sh` |
|------------|-----|---------------------------|
| **xPack GNU RISC-V GCC 15.2.0-1** (`riscv-none-elf`) | newlib + libstdc++; `-march=rv64imfd_zicsr` (no A, no C) | `$TOOLS_DIR/xpack-riscv-none-elf-gcc-15.2.0-1` |
| **CMake 3.30.5** (portable) | llama.cpp is CMake; ≥ 3.14 required | `$TOOLS_DIR/cmake-3.30.5-linux-x86_64` |
| **numpy** (Python venv) | optional dummy GGUF via `gen_tiny_gguf.py` | `$TOOLS_DIR/cva6-gguf-venv` |

Also needed on the host (install with apt if missing; `setup.sh` does not):

| Host package | Used for |
|-------------|---------|
| `curl`, `tar`, `coreutils` | download and unpack the toolchains |
| `python3`, `python3-venv` | optional dummy GGUF |
| `make` | sle-benchmarks `tests/llama/Makefile` |
| [sle-benchmarks](https://github.com/SilverLining-EDA/sle-benchmarks) | `tests/llama` glue, `llama.bin`, `run.sh` |
| [aws-fpga](https://github.com/SilverLining-EDA/aws-fpga) | F2 SDK + [`cl_cva6_llama`](https://github.com/SilverLining-EDA/aws-fpga/tree/main/hdk/cl/examples/cl_cva6_llama) interactive loader |

Default `TOOLS_DIR` is `/projects/prj1/sle-wajahat/tools` on the F2 instance, otherwise `../tools` next to this clone. Override with `TOOLS_DIR=/path ./setup.sh`.

### 1. Clone this fork

```bash
git clone git@github.com:SilverLining-EDA/llama.cpp.git
cd llama.cpp
```

### 2. Install toolchains

```bash
chmod +x setup.sh
./setup.sh
source /projects/prj1/sle-wajahat/tools/cva6-env.sh
# if TOOLS_DIR was not the F2 default:
# source "$TOOLS_DIR/cva6-env.sh"
```

This sets `XPACK_ROOT` and `CMAKE` for the benchmark Makefile.

### 3. Clone the benchmark harness

```bash
git clone git@github.com:SilverLining-EDA/sle-benchmarks.git
export LLAMA_SRC="$(pwd)"   # this llama.cpp tree
```

### 4. Get a GGUF

`model.gguf` is not in git. `make` in `tests/llama` downloads **TinyStories 15M Q8** from `ggml-org/models-moved` if the file is missing (the older `tinyllamas` Hugging Face path often returns 401).

```bash
cd /path/to/sle-benchmarks/tests/llama
curl -L --fail -o model.gguf \
  "https://huggingface.co/ggml-org/models-moved/resolve/main/tinyllamas/stories15M-q8_0.gguf"
```

Prompt **`Once upon a time`**. Dummy `gen_tiny_gguf.py` weights are random and will not produce English.

### 5. Build `llama.bin`

```bash
source /projects/prj1/sle-wajahat/tools/cva6-env.sh
cd /path/to/sle-benchmarks/tests/llama
make LLAMA_SRC=/path/to/llama.cpp
# on the F2 layout the Makefile defaults already match
```

March is `rv64imfd_zicsr` / `lp64d`. Do not use `rv64gc`.

### 6. Interactive run on F2

AGFI `agfi-0248c1f84010b03e9`. Host loader: [`cl_cva6_llama`](https://github.com/SilverLining-EDA/aws-fpga/tree/main/hdk/cl/examples/cl_cva6_llama) (logo + stdin prompts). UART is CVA6→host only; each prompt is an HBM mailbox write while CVA6 is in reset.

```bash
export AWS_FPGA_REPO_DIR=/projects/prj1/sle-wajahat/aws-fpga
cd /path/to/sle-benchmarks/tests/llama
./run.sh
# SKIP_AGFI_LOAD=1 ./run.sh   # AFI already on the slot
```

Wait until CVA6 prints `>>> `, then type **`Once upon a time`**. `/bye` quits.

Each prompt **resets CVA6 and reloads the GGUF** (this AGFI cannot write HBM while the CPU runs). The host rewrites `llama.bin` on every prompt so `.data` in HBM is clean. First load ~20–40 s; then ~10 s/token.

This HBM port **never completes AMO/LR/SC**; that is why the image is built without the A extension. SPM vocabs that omit `<0xXX>` must not use `unordered_map::at()` (bare-metal exception unwind hangs).

![llama](https://raw.githubusercontent.com/ggml-org/llama.brand/refs/heads/master/cover/llama-cpp/cover-llama-cpp-dark.svg)

<div align="center">

<b>LLM inference in C/C++</b>

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![Release](https://img.shields.io/github/v/release/ggml-org/llama.cpp?filter=v*&color=brightgreen)](https://github.com/ggml-org/llama.cpp/releases?q=tag:v0)
[![Nightly](https://img.shields.io/github/v/release/ggml-org/llama.cpp?label=nightly&filter=b*&color=orange)](https://github.com/ggml-org/llama.cpp/releases?q=b)
[![Server](https://img.shields.io/github/actions/workflow/status/ggml-org/llama.cpp/server.yml?label=Server)](https://github.com/ggml-org/llama.cpp/actions/workflows/server.yml)
[![Docker](https://img.shields.io/github/actions/workflow/status/ggml-org/llama.cpp/docker.yml?label=Docker)](https://github.com/ggml-org/llama.cpp/actions/workflows/docker.yml)
[![Winget](https://img.shields.io/github/actions/workflow/status/ggml-org/llama.cpp/winget.yml?label=Winget)](https://github.com/ggml-org/llama.cpp/actions/workflows/winget.yml)

[ggml](https://github.com/ggml-org/ggml) / [ops](https://github.com/ggml-org/llama.cpp/blob/master/docs/ops.md) / [maintainer PRs](https://github.com/ggml-org/llama.cpp/issues?q=is%3Apr%20is%3Aopen%20draft%3AFalse%20(author%3Argerganov%20OR%20author%3AKitaitiMakoto%20OR%20author%3Adanbev%20OR%20author%3Aaldehir%20OR%20author%3Amax-krasnyansky%20OR%20author%3ACISC%20OR%20author%3Aggerganov%20OR%20author%3Aam17an%20OR%20author%3Abartowski1182%20OR%20author%3Anikwen%20OR%20author%3Ahipudding%20OR%20author%3AServeurpersoCom%20OR%20author%3Apwilkin%20OR%20author%3Areeselevine%20OR%20author%3Angxson%20OR%20author%3Ajeffbolznv%20OR%20author%3Amarty1885%20OR%20author%3A0cc4m%20OR%20author%3ATitaniumtown%20OR%20author%3Aangt%20OR%20author%3AIMbackK%20OR%20author%3Aarthw%20OR%20author%3AJohannesGaessler%20OR%20author%3AORippler%20OR%20author%3Aruixiang63%20OR%20author%3Axctan%20OR%20author%3Aallozaur%20OR%20author%3Ayomaytk%20OR%20author%3Aaendk%20OR%20author%3Agaugarg-nv%20OR%20author%3Ataronaeo%20OR%20author%3Aforforever73%20OR%20author%3Alhez%20OR%20author%3Anetrunnereve%20OR%20author%3Afairydreaming)%20sort%3Aupdated-desc) / [dev stats](https://github.com/ggml-org/llama.cpp-dev) / [lib llama API](https://github.com/ggml-org/llama.cpp/issues/9289) / [llama-server REST API](https://github.com/ggml-org/llama.cpp/issues/9291)

</div>

## Quick start

A few options to get `llama.cpp` installed on your machine:

- Visit https://llama.app and follow the instructions
- Run with Docker - see our [Docker documentation](docs/docker.md)
- Download pre-built binaries from the [releases page](https://github.com/ggml-org/llama.cpp/releases)
- Build from source by cloning this repository - check out [our build guide](docs/build.md)

Once installed:

```sh
# Download and run a model directly from Hugging Face
llama cli -hf ggml-org/Qwen3.5-0.8B-GGUF

# Launch OpenAI-compatible API server
llama serve -hf ggml-org/Qwen3.5-0.8B-GGUF
```

<table align="center">
    <tr>
        <td align="center" width=50%>
            <img width="1310" height="888" alt="VLM session with `llama cli`" src="https://github.com/user-attachments/assets/88726b48-1713-48aa-a525-95a02e78afc4" />
            <i>VLM session with <b>llama cli</b></i>
        </td>
        <td align="center">
            <img width="1392" height="958" alt="Built-in web UI against `llama serve` running Qwen 3.6" src="https://github.com/user-attachments/assets/b402f972-2e32-4def-8771-8d849f08cf2e" />
            <i>Built-in web UI against <b>llama serve</b></i>
        </td>
    </tr>
<table>

## Description

The main goal of `llama.cpp` is to enable LLM (and VLM) inference with minimal setup and state-of-the-art performance on
a wide range of hardware - locally and in the cloud.

- Plain C/C++ implementation without any dependencies
- Apple silicon is a first-class citizen - optimized via ARM NEON, Accelerate and Metal frameworks
- AVX, AVX2, AVX512 and AMX support for x86 architectures
- RVV, ZVFH, ZFH, ZICBOP and ZIHINTPAUSE support for RISC-V architectures
- 1.5-bit, 2-bit, 3-bit, 4-bit, 5-bit, 6-bit, and 8-bit integer quantization for faster inference and reduced memory use
- Custom CUDA kernels for running LLMs on NVIDIA GPUs (support for AMD GPUs via HIP and Moore Threads GPUs via MUSA)
- Vulkan and SYCL backend support
- CPU+GPU hybrid inference to partially accelerate models larger than the total VRAM capacity

The `llama.cpp` project is build on top of the [ggml](https://github.com/ggml-org/ggml) library.

## Supported backends

| Backend | Target devices |
| --- | --- |
| [BLAS](docs/build.md#blas-build) | All |
| [BLIS](docs/backend/BLIS.md) | All |
| [CANN](docs/build.md#cann) | Ascend NPU |
| [CUDA](docs/build.md#cuda) | Nvidia GPU |
| [HIP](docs/build.md#hip) | AMD GPU |
| [Hexagon [In Progress]](docs/backend/snapdragon/README.md) | Snapdragon |
| [IBM zDNN](docs/backend/zDNN.md) | IBM Z & LinuxONE |
| [MUSA](docs/build.md#musa) | Moore Threads GPU |
| [Metal](docs/build.md#metal-build) | Apple Silicon |
| [OpenCL](docs/backend/OPENCL.md) | Adreno GPU |
| [OpenVINO [In Progress]](docs/backend/OPENVINO.md) | Intel CPUs, GPUs, and NPUs |
| [RPC](https://github.com/ggml-org/llama.cpp/tree/master/tools/rpc) | All |
| [SYCL](docs/backend/SYCL.md) | Intel GPU |
| [VirtGPU](docs/backend/VirtGPU.md) | VirtGPU APIR |
| [Vulkan](docs/build.md#vulkan) | GPU |
| [WebGPU](docs/build.md#webgpu) | All |
| [ZenDNN](docs/build.md#zendnn) | AMD CPU |

## Documentation

#### Tools

- [cli](tools/cli/README.md)
- [completion](tools/completion/README.md)
- [server](tools/server/README.md)
- [GBNF grammars](grammars/README.md)

#### Development

- [How to build](docs/build.md)
- [Running on Docker](docs/docker.md)
- [Build on Android](docs/android.md)
- [Multi-GPU usage](docs/multi-gpu.md)
- [Performance troubleshooting](docs/development/token_generation_performance_tips.md)
- [GGML tips & tricks](https://github.com/ggml-org/llama.cpp/wiki/GGML-Tips-&-Tricks)
- [XCFramework](docs/xcframework.md)
- [Completions](docs/completions.md)
- [Models](docs/models.md)
- [Release process](docs/release.md)

## Contributing

- Contributors can open PRs
- Collaborators will be invited based on contributions
- Maintainers can push to branches in the `llama.cpp` repo and merge PRs into the `master` branch
- Any help with managing issues, PRs and projects is very appreciated!
- Read the [CONTRIBUTING.md](CONTRIBUTING.md) for more information

## Acknowledgements

- [yhirose/cpp-httplib](https://github.com/yhirose/cpp-httplib) - Single-header HTTP server, used by `llama-server` - MIT license
- [nothings/stb](https://github.com/nothings/stb) - Single-header image format decoder, used by multimodal subsystem - Public domain
- [nlohmann/json](https://github.com/nlohmann/json) - Single-header JSON library, used by various tools/examples - MIT License
- [mackron/miniaudio](https://github.com/mackron/miniaudio) - Single-header audio format decoder, used by multimodal subsystem - Public domain
- [sheredom/subprocess.h](https://github.com/sheredom/subprocess.h) - Single-header process launching solution for C and C++ - Public domain
