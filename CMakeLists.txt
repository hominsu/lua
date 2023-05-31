CMAKE_MINIMUM_REQUIRED(VERSION 2.8.12)
if (POLICY CMP0025)
    # detect Apple's Clang
    cmake_policy(SET CMP0025 NEW)
endif ()
if (POLICY CMP0040)
    cmake_policy(SET CMP0040 NEW)
endif ()
if (POLICY CMP0042)
    cmake_policy(SET CMP0042 NEW)
endif ()
if (POLICY CMP0054)
    cmake_policy(SET CMP0054 NEW)
endif ()

set(LIB_MAJOR_VERSION "5")
set(LIB_MINOR_VERSION "4")
set(LIB_PATCH_VERSION "6")
set(LIB_VERSION_STRING "${LIB_MAJOR_VERSION}.${LIB_MINOR_VERSION}.${LIB_PATCH_VERSION}")

if (CMAKE_VERSION VERSION_LESS 3.0)
    PROJECT(lua C)
else ()
    cmake_policy(SET CMP0048 NEW)
    PROJECT(lua VERSION "${LIB_VERSION_STRING}" LANGUAGES C)
endif ()

SET(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
SET(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
SET(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)

option(LUA_ENABLE_INSTRUMENTATION_OPT "Build Lua with -march or -mcpu options" ON)

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED TRUE)

message("")
message("Operation system is ${CMAKE_SYSTEM}")
message("Current compiler: ${CMAKE_C_COMPILER_ID}")
message("Current compiler version: ${CMAKE_C_COMPILER_VERSION}")
message("Current compiler directory: ${CMAKE_C_COMPILER}")

if (CMAKE_C_COMPILER_ID STREQUAL "GNU")
    if (LUA_ENABLE_INSTRUMENTATION_OPT AND NOT CMAKE_CROSSCOMPILING)
        if (CMAKE_SYSTEM_PROCESSOR STREQUAL "powerpc" OR CMAKE_SYSTEM_PROCESSOR STREQUAL "ppc" OR CMAKE_SYSTEM_PROCESSOR STREQUAL "ppc64" OR CMAKE_SYSTEM_PROCESSOR STREQUAL "ppc64le")
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mcpu=native")
        else ()
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -march=native")
        endif ()
    endif ()
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -fno-stack-protector -fno-common")
elseif (CMAKE_C_COMPILER_ID MATCHES "Clang")
    if (LUA_ENABLE_INSTRUMENTATION_OPT AND NOT CMAKE_CROSSCOMPILING)
        if (CMAKE_SYSTEM_PROCESSOR STREQUAL "powerpc" OR CMAKE_SYSTEM_PROCESSOR STREQUAL "ppc" OR CMAKE_SYSTEM_PROCESSOR STREQUAL "ppc64" OR CMAKE_SYSTEM_PROCESSOR STREQUAL "ppc64le")
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mcpu=native")
        else ()
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -march=native")
        endif ()
    endif ()
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -fno-stack-protector -fno-common")
endif ()

add_subdirectory(src)