cmake_minimum_required(VERSION 3.10)

if(NOT DEFINED CMAKE_BUILD_TYPE OR CMAKE_BUILD_TYPE STREQUAL "")
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build, options are: Debug, Release, RelWithDebInfo" FORCE)
endif()

message(STATUS "Mode: ${CMAKE_BUILD_TYPE}")

set(ARCH native CACHE STRING "CPU to build for: -march value or native")

message(STATUS "Building for target architecture: ${ARCH}")

project(argon2 VERSION "1.0.2" LANGUAGES C)

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

if(MSVC)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /D_CRT_SECURE_NO_WARNINGS")
else()
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wuninitialized")

    set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -g3 -Og")

    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -O3")

    if(MINGW)
        set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -static")
    endif()
endif()

message(STATUS "Processor: ${CMAKE_SYSTEM_PROCESSOR}")

if(NOT "${ARCH}" STREQUAL "default")
    include(OptimizeForArchitecture)
    OptimizeForArchitecture()

    if(MSVC)
        # Needed by MSVC, but not added by OptimizeForArchitecture()
        add_definitions(-D__SSE__)
    endif()

    foreach(flag ${Vc_ARCHITECTURE_FLAGS})
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${flag}")
    endforeach()
endif()

message(STATUS "Build C Flags: ${CMAKE_C_FLAGS}")

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

add_library(argon2 STATIC ${ARGON2_SRC})
target_include_directories(argon2
    PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include
    PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/lib ${CMAKE_CURRENT_SOURCE_DIR}/lib/blake2
)
target_link_libraries(argon2 Threads::Threads)
set_property(TARGET argon2 PROPERTY C_STANDARD 90)
set_property(TARGET argon2 PROPERTY C_STANDARD_REQUIRED ON)
