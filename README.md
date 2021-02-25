# Argon2 [![Build Status](https://travis-ci.org/WOnder93/argon2.svg?branch=master)](https://travis-ci.org/WOnder93/argon2)
A multi-arch library implementing the Argon2 password hashing algorithm.

This project is based on the [original source code](https://github.com/P-H-C/phc-winner-argon2) by the Argon2 authors. The goal of this project is to provide efficient Argon2 implementations for various HW architectures (x86, SSE, ARM, PowerPC, ...).

For the x86_64 architecture, the library implements a simple CPU dispatch which automatically selects the best implementation based on CPU flags and quick benchmarks.

## Using CMake

To prepare the build environment, run:
```bash
cmake -DCMAKE_BUILD_TYPE=Release .
```

Then you can run `make` to build the library.

### Architecture Support

**Note:** The CMake build system will autodetect the architecture options and intrinsics supported by your compiler. Auto selection of the fastest methods can be accomplished using:

```c
argon2_select_impl(NULL, NULL);
```

Supported architectures:
 * `x86_64` &ndash; 64-bit x86 architecture
   * QMake config flags:
     * `USE_SSE2` &ndash; use SSE2 instructions
     * `USE_SSSE3` &ndash; use SSSE3 instructions
     * `USE_XOP` &ndash; use XOP instructions
     * `USE_AVX2` &ndash; use AVX2 instructions
     * `USE_AVX512F` &ndash; use AVX-512F instructions
 * `generic` &ndash; use generic C impementation
