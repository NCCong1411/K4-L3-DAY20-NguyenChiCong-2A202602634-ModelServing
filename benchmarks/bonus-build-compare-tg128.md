# Bonus B1 - Prebuilt vs source build

Host `Windows-AMD64` · CPU `AMD Ryzen 5 7535HS with Radeon Graphics`
Vector extensions used by the native build: AVX2, FMA, F16C (from CMake configure log)
llama.cpp `b10488` both sides · `threads=6` ·
**both pinned to `ngl=0`** so this isolates the compiler ·
metric `tg128`, 3 repetitions

| Binary | Built for | tg128 (tok/s) | Relative |
|:--|--:|--:|--:|
| prebuilt release | runtime CPU dispatch | 18.2 | 1.00x |
| your source build | this CPU (`-DGGML_NATIVE=ON`) | 17.3 | 0.95x |

On this machine, the prebuilt binary is **1.05x faster**.

before: 18.2 tok/s (prebuilt release)
after:  17.3 tok/s (source build, -DGGML_NATIVE=ON)
speedup: 0.95x

Same source revision, same model, same backend, same `-ngl` -- the only difference
is what the compiler was allowed to assume about the CPU.



## My explanation

CMake compiled the native CPU backend with `/arch:AVX2` and enabled AVX2, FMA and
F16C. Even so, the prebuilt release was 1.05x faster (18.2 versus 17.3 tok/s). This
is plausible because the prebuilt does runtime CPU dispatch and can already select
an AVX2 kernel on this Ryzen; `-DGGML_NATIVE=ON` therefore did not unlock a wider
instruction set that the release lacked.

The `tg128` decode workload repeatedly streams the model weights, so shared memory
bandwidth and cache behaviour matter more than peak arithmetic throughput. Once both
binaries use comparable vector kernels, compiler/code-generation differences and
normal run-to-run variation can outweigh any small native-build benefit. The honest
conclusion is that rebuilding is not a performance win on this machine; the prebuilt
is both simpler and about 5% faster in this measurement.
