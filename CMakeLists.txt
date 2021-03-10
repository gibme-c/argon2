cmake_minimum_required(VERSION 2.8.12)

find_program(CCACHE_PROGRAM ccache)
if(CCACHE_PROGRAM)
    message(STATUS "ARGON2: Found ccache package... Activating...")
    set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE "${CCACHE_PROGRAM}")
endif()

set(LIB_MAJOR_VERSION "1")
set(LIB_MINOR_VERSION "0")
set(LIB_PATCH_VERSION "0")
set(LIB_VERSION_STRING "${LIB_MAJOR_VERSION}.${LIB_MINOR_VERSION}.${LIB_PATCH_VERSION}")

if(CMAKE_VERSION VERSION_LESS 3.0)
    project(argon2 C)
    enable_language(C)
    enable_language(CXX)
else()
    cmake_policy(SET CMP0048 NEW)
    project(argon2 VERSION "${LIB_VERSION_STRING}" LANGUAGES C CXX)
endif()

## This section describes our general CMake setup options
set_property(GLOBAL PROPERTY USE_FOLDERS ON)
set(CMAKE_SKIP_INSTALL_RULES OFF FORCE)
set(CMAKE_SKIP_PACKAGE_ALL_DEPENDENCY ON FORCE)
set(CMAKE_SUPPRESS_REGENERATION ON)
set(CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake" ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

## We only build static binaries -- this is left here for our dependencies
set(STATIC ON CACHE BOOL FORCE "Link libraries statically? Forced to ON")
add_definitions(-DSTATICLIB)

set(ARGON2_VERSION 1.0)

# Enable c++11
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

include(CheckCSourceCompiles)
find_package(Threads REQUIRED)

if(MSVC)
    add_definitions("/EHsc -std:c++17 /DWIN32_LEAN_AND_MEAN /O2 /wd4267 /D_CRT_SECURE_NO_WARNINGS")
endif()

if(NOT MSVC)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=c11 -O3")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++17 -O3")
endif()

set(ARCH native CACHE STRING "CPU to build for: -march value or native")

message("-- Argon2: Processor: ${CMAKE_SYSTEM_PROCESSOR}")
message("-- Argon2: Build Type: ${ARCH}")

if(NOT "${ARCH}" STREQUAL "default")
    include(OptimizeForArchitecture)
    OptimizeForArchitecture()

    if (MSVC)
        # Needed by MSVC, but not added by OptimizeForArchitexture()
        add_definitions(-D__SSE__)
    endif()
endif()

foreach(flag ${Vc_ARCHITECTURE_FLAGS})
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${flag}")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${flag}")
endforeach()

message(STATUS "-- Argon2: Build CXX Flags: ${CMAKE_CXX_FLAGS}")
message(STATUS "-- Argon2: Build C Flags: ${CMAKE_C_FLAGS}")

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
target_include_directories(argon2 PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include ${CMAKE_CURRENT_SOURCE_DIR}/lib ${CMAKE_CURRENT_SOURCE_DIR}/lib/blake2)
target_link_libraries(argon2 ${CMAKE_THREAD_LIBS_INIT})
target_compile_definitions(argon2 PUBLIC "A2_VISCTL")

set_property(TARGET argon2 PROPERTY C_STANDARD 90)
set_property(TARGET argon2 PROPERTY C_STANDARD_REQUIRED ON)
set_property(TARGET argon2 PROPERTY SOVERSION 1)

foreach(flag_var
        CMAKE_C_FLAGS CMAKE_C_FLAGS_DEBUG CMAKE_C_FLAGS_RELEASE
        CMAKE_C_FLAGS_MINSIZEREL CMAKE_C_FLAGS_RELWITHDEBINFO
        CMAKE_CXX_FLAGS CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE
        CMAKE_CXX_FLAGS_MINSIZEREL CMAKE_CXX_FLAGS_RELWITHDEBINFO)
    string(REGEX REPLACE "/RTC[^ ]*" "" ${flag_var} "${${flag_var}}")
    string(REGEX REPLACE "/Od" "" ${flag_var} "${${flag_var}}")
endforeach(flag_var)
