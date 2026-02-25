cmake_minimum_required(VERSION 3.10)

if(NOT DEFINED CMAKE_BUILD_TYPE OR CMAKE_BUILD_TYPE STREQUAL "")
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build, options are: Debug, Release, RelWithDebInfo" FORCE)
endif()

message(STATUS "Mode: ${CMAKE_BUILD_TYPE}")

set(ARCH native CACHE STRING "CPU to build for: -march value or native")

message(STATUS "Building for target architecture: ${ARCH}")

project(argon2 VERSION "1.0.2" LANGUAGES C)

include(CheckCCompilerFlag)
include(CheckCSourceCompiles)

if(NOT MSVC)
    find_program(CCACHE_PROGRAM ccache)
    if(CCACHE_PROGRAM)
        message(STATUS "Found ccache package... Activating...")
        set(CMAKE_C_COMPILER_LAUNCHER "${CCACHE_PROGRAM}")
    endif()
endif()

set_property(GLOBAL PROPERTY USE_FOLDERS ON)
set(CMAKE_SUPPRESS_REGENERATION ON)
set(CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake" ${CMAKE_MODULE_PATH})

find_package(Threads REQUIRED)

message(STATUS "Processor: ${CMAKE_SYSTEM_PROCESSOR}")

# --- Architecture detection for SIMD ---
if(NOT "${ARCH}" STREQUAL "default")
    include(OptimizeForArchitecture)
    OptimizeForArchitecture()
endif()

# --- Source files ---
set(ARGON2_SRC
    lib/argon2.c
    lib/core.c
    lib/encoding.c
    lib/genkat.c
    lib/impl-select.c
    lib/thread.c
    lib/blake2/blake2.c
)

if(USE_AVX512F OR USE_AVX2 OR USE_SSE3 OR USE_SSE2 OR USE_XOP)
    list(APPEND ARGON2_SRC
        arch/x86_64/lib/argon2-sse2.c
        arch/x86_64/lib/argon2-sse3.c
        arch/x86_64/lib/argon2-xop.c
        arch/x86_64/lib/argon2-avx2.c
        arch/x86_64/lib/argon2-avx512f.c
        arch/x86_64/lib/argon2-arch.c
    )
else()
    list(APPEND ARGON2_SRC
        arch/generic/lib/argon2-arch.c
    )
endif()

# --- Library target ---
add_library(argon2 STATIC ${ARGON2_SRC})

target_include_directories(argon2
    PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include
    PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/lib ${CMAKE_CURRENT_SOURCE_DIR}/lib/blake2
)

target_link_libraries(argon2 PRIVATE Threads::Threads)

set_property(TARGET argon2 PROPERTY C_STANDARD 90)
set_property(TARGET argon2 PROPERTY C_STANDARD_REQUIRED ON)

# --- Architecture-specific compile flags (PRIVATE) ---
if(NOT "${ARCH}" STREQUAL "default")
    if(MSVC)
        # Needed by MSVC, but not added by OptimizeForArchitecture()
        target_compile_definitions(argon2 PRIVATE __SSE__)
    endif()
    foreach(flag ${Vc_ARCHITECTURE_FLAGS})
        target_compile_options(argon2 PRIVATE ${flag})
    endforeach()
endif()

# --- MinGW SIMD stack alignment fix ---
# Windows x64 ABI only guarantees 16-byte stack alignment, but YMM/ZMM registers
# need 32/64-byte alignment. At -O0, GCC spills AVX registers to the stack with
# aligned moves (vmovdqa), which SIGSEGV on the misaligned stack. Force -O1 for
# SIMD backends in Debug mode so GCC keeps values in registers.
if(MINGW AND CMAKE_C_COMPILER_ID STREQUAL "GNU")
    if(USE_AVX512F OR USE_AVX2)
        set(_MINGW_SIMD_FIX "$<$<CONFIG:Debug>:-O1>")
        set_source_files_properties(
            arch/x86_64/lib/argon2-avx2.c
            arch/x86_64/lib/argon2-avx512f.c
            PROPERTIES COMPILE_FLAGS "${_MINGW_SIMD_FIX}"
        )
    endif()
endif()

# --- Detect MinGW environment ---
# Only use CMake's built-in MINGW (set for GCC with MinGW). Clang on Windows can
# target either MinGW or MSVC ABI, and the linker varies (GNU ld vs lld-link) —
# applying the wrong linker flags breaks the build. We let Clang on Windows rely
# on the OS defaults for DEP/ASLR (enabled by default on modern Windows) rather
# than guessing the linker style.

# --- Hardening flags (all PRIVATE — never leak to downstream consumers) ---
if(CMAKE_C_COMPILER_ID MATCHES "GNU|Clang")
    target_compile_options(argon2 PRIVATE
        -Wall -Wextra
        -Wno-type-limits
        -Wcast-align -Wundef
        -Wformat=2 -Wformat-security
        -fno-strict-overflow
    )
    target_compile_options(argon2 PRIVATE
        $<$<CONFIG:Debug>:-g3 -Og>
        $<$<CONFIG:Release>:-O3>
    )
    # Stack protector: requires libssp or compiler-rt at link time.
    # Probe with try_compile to catch Clang+MinGW toolchains that lack libssp.
    set(CMAKE_REQUIRED_FLAGS "-fstack-protector-strong")
    check_c_source_compiles("int main(void) { char buf[64]; buf[0] = 0; return buf[0]; }" HAS_STACK_PROTECTOR_STRONG)
    unset(CMAKE_REQUIRED_FLAGS)
    if(HAS_STACK_PROTECTOR_STRONG)
        target_compile_options(argon2 PRIVATE -fstack-protector-strong)
    endif()
    # -fPIC is implied on Windows PE; only needed on ELF/Mach-O
    if(NOT WIN32)
        target_compile_options(argon2 PRIVATE -fPIC)
        # Stack clash protection (Linux only — not supported on macOS/Apple Clang)
        if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
            target_compile_options(argon2 PRIVATE -fstack-clash-protection)
        endif()
    endif()
    # Intel CET control flow integrity (x86_64 only)
    if(CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64|AMD64")
        check_c_compiler_flag("-fcf-protection=full" HAS_CF_PROTECTION)
        if(HAS_CF_PROTECTION)
            target_compile_options(argon2 PRIVATE -fcf-protection=full)
        endif()
    endif()
    # -Wstrict-aliasing=2 only for GCC (Clang doesn't support the =2 level)
    if(CMAKE_C_COMPILER_ID STREQUAL "GNU")
        target_compile_options(argon2 PRIVATE -Wstrict-aliasing=2)
    endif()
    # _FORTIFY_SOURCE requires optimization; not reliably supported on MinGW
    if(NOT WIN32)
        target_compile_options(argon2 PRIVATE
            $<$<NOT:$<CONFIG:Debug>>:-D_FORTIFY_SOURCE=2>
        )
    endif()
    # Linker hardening
    if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
        set_property(TARGET argon2 APPEND_STRING PROPERTY LINK_FLAGS
            " -Wl,-z,relro,-z,now -Wl,-z,noexecstack")
    endif()
    # MinGW linker hardening (DEP + ASLR) — only for GCC+MinGW (known GNU ld)
    if(MINGW)
        set_property(TARGET argon2 APPEND_STRING PROPERTY LINK_FLAGS
            " -Wl,--nxcompat -Wl,--dynamicbase -Wl,--high-entropy-va")
    endif()
elseif(MSVC)
    target_compile_options(argon2 PRIVATE
        /W4
        /sdl
        /GS
        /guard:cf
    )
    target_compile_definitions(argon2 PRIVATE _CRT_SECURE_NO_WARNINGS)
    check_c_compiler_flag("/Qspectre" HAS_QSPECTRE)
    if(HAS_QSPECTRE)
        target_compile_options(argon2 PRIVATE /Qspectre)
    endif()
    set_property(TARGET argon2 APPEND_STRING PROPERTY LINK_FLAGS
        " /DYNAMICBASE /NXCOMPAT /HIGHENTROPYVA")
    # CET shadow stack compatibility
    check_c_compiler_flag("/guard:ehcont" HAS_GUARD_EHCONT)
    if(HAS_GUARD_EHCONT)
        target_compile_options(argon2 PRIVATE /guard:ehcont)
        set_property(TARGET argon2 APPEND_STRING PROPERTY LINK_FLAGS " /guard:ehcont /CETCOMPAT")
    endif()
endif()
