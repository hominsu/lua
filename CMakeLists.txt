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
    PROJECT(lua C CXX)
else ()
    cmake_policy(SET CMP0048 NEW)
    PROJECT(lua VERSION "${LIB_VERSION_STRING}" LANGUAGES C CXX)
endif ()

# compile in release with debug info mode by default
if (NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel." FORCE)
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
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wfatal-errors -Wextra")
    set(EXTRA_C_FLAGS
            -Wshadow -Wundef -Wwrite-strings -Wredundant-decls -Wdisabled-optimization -Wdouble-promotion -Wmissing-declarations
            -Wdeclaration-after-statement -Wmissing-prototypes -Wnested-externs -Wstrict-prototypes -Wc++-compat -Wold-style-definition
            -Wlogical-op -Wno-aggressive-loop-optimizations
            -Wall -O2 -fno-stack-protector -fno-common)
elseif (CMAKE_C_COMPILER_ID MATCHES "Clang")
    if (LUA_ENABLE_INSTRUMENTATION_OPT AND NOT CMAKE_CROSSCOMPILING)
        if (CMAKE_SYSTEM_PROCESSOR STREQUAL "powerpc" OR CMAKE_SYSTEM_PROCESSOR STREQUAL "ppc" OR CMAKE_SYSTEM_PROCESSOR STREQUAL "ppc64" OR CMAKE_SYSTEM_PROCESSOR STREQUAL "ppc64le")
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mcpu=native")
        else ()
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -march=native")
        endif ()
    endif ()
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wfatal-errors -Wextra")
    set(EXTRA_C_FLAGS
            -Wshadow -Wundef -Wwrite-strings -Wredundant-decls -Wdisabled-optimization -Wdouble-promotion -Wmissing-declarations
            -Wdeclaration-after-statement -Wmissing-prototypes -Wnested-externs -Wstrict-prototypes -Wc++-compat -Wold-style-definition
            -Wall -O2 -fno-stack-protector -fno-common)
endif ()

# core: used by all
file(GLOB CORE_SRCS
        src/lzio.c
        src/lctype.c
        src/lopcodes.c
        src/lmem.c
        src/lundump.c
        src/ldump.c
        src/lstate.c
        src/lgc.c
        src/llex.c
        src/lcode.c
        src/lparser.c
        src/ldebug.c
        src/lfunc.c
        src/lobject.c
        src/ltm.c
        src/lstring.c
        src/ltable.c
        src/ldo.c
        src/lvm.c
        src/lapi.c
        )

# auxiliary library: used by all
file(GLOB AUX_SRCS
        src/lauxlib.c
        )

# standard library: not used by luac
file(GLOB LIB_SRCS
        src/lbaselib.c
        src/lcorolib.c
        src/ldblib.c
        src/liolib.c
        src/lmathlib.c
        src/loadlib.c
        src/loslib.c
        src/lstrlib.c
        src/ltablib.c
        src/lutf8lib.c
        src/linit.c
        )

# lua
file(GLOB LUA_SRCS
        src/lua.c
        )

add_library(lua_include INTERFACE)
target_include_directories(lua_include INTERFACE
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}>
        $<INSTALL_INTERFACE:include/${PROJECT_NAME}>)

add_library(lua_lib STATIC ${CORE_SRCS} ${AUX_SRCS} ${LIB_SRCS})
target_link_libraries(lua_lib PUBLIC lua_include)
if (UNIX)
    find_library(LIBM m)
    if (NOT LIBM)
        message(FATAL_ERROR "libm not found and is required by lua")
    endif ()
    if (APPLE)
        set(_LUA_PLATFORM LUA_USE_MACOSX)
    else ()
        set(_LUA_PLATFORM LUA_USE_LINUX)
    endif ()
    target_compile_definitions(lua_lib INTERFACE ${_LUA_PLATFORM})
    target_link_libraries(lua_lib PUBLIC m dl)
endif ()
set_target_properties(lua_lib PROPERTIES
        COMPILE_OPTIONS ${EXTRA_C_FLAGS}
        VERSION "${PACKAGE_VERSION}"
        OUTPUT_NAME lua)

add_library(Lua::Lua ALIAS lua_lib)

add_executable(lua ${LUA_SRCS})
target_link_libraries(lua PUBLIC lua_include PRIVATE lua_lib)
include(CheckIncludeFile)
CHECK_INCLUDE_FILE("readline/readline.h" HAVE_READLINE)
if (HAVE_READLINE)
    target_compile_definitions(lua PRIVATE LUA_USE_READLINE)
    target_link_libraries(lua PUBLIC readline)
endif ()
set_target_properties(lua PROPERTIES
        COMPILE_OPTIONS ${EXTRA_C_FLAGS})

#add extra search paths for libraries and includes
SET(INCLUDE_INSTALL_DIR "${CMAKE_INSTALL_PREFIX}/include" CACHE PATH "The directory the headers are installed in")
SET(LIB_INSTALL_DIR "${CMAKE_INSTALL_PREFIX}/lib" CACHE STRING "Directory where lib will install")

# cmake
if (UNIX OR CYGWIN)
    set(_CMAKE_INSTALL_DIR "${LIB_INSTALL_DIR}/cmake/${PROJECT_NAME}")
elseif (WIN32)
    set(_CMAKE_INSTALL_DIR "${CMAKE_INSTALL_PREFIX}/cmake")
endif ()
set(CMAKE_INSTALL_DIR "${_CMAKE_INSTALL_DIR}" CACHE PATH "The directory cmake files are installed in")

# pkg-config
if (UNIX OR CYGWIN)
    configure_file(
            ${CMAKE_CURRENT_SOURCE_DIR}/${PROJECT_NAME}.pc.in
            ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.pc
            @ONLY)
    install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.pc
            DESTINATION "${LIB_INSTALL_DIR}/pkgconfig"
            COMPONENT pkgconfig)
endif ()

install(TARGETS lua_include lua_lib
        EXPORT LuaTargets)

install(DIRECTORY include/${PROJECT_NAME}
        DESTINATION "${INCLUDE_INSTALL_DIR}")

install(EXPORT LuaTargets
        FILE LuaTargets.cmake
        DESTINATION ${CMAKE_INSTALL_DIR}
        NAMESPACE Lua::)

string(TOUPPER ${PROJECT_NAME} _PROJECT_NAME)

get_target_property(LUA_EXPORT_LIBRARY Lua::Lua ALIASED_TARGET)
set(${_PROJECT_NAME}_INCLUDE_DIR "\${${_PROJECT_NAME}_SOURCE_DIR}/include")
set(CONFIG_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR})
set(CONFIG_DIR ${CMAKE_CURRENT_BINARY_DIR})
configure_file(
        "${CMAKE_CURRENT_SOURCE_DIR}/${PROJECT_NAME}Config.cmake.in"
        "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
        @ONLY)
configure_file(
        "${CMAKE_CURRENT_SOURCE_DIR}/${PROJECT_NAME}ConfigVersion.cmake.in"
        "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
        @ONLY)

set(CMAKECONFIG_INSTALL_DIR ${LIB_INSTALL_DIR}/cmake/${PROJECT_NAME})
file(RELATIVE_PATH REL_INCLUDE_DIR
        "${CMAKECONFIG_INSTALL_DIR}"
        "${CMAKE_INSTALL_PREFIX}/include")
set(${_PROJECT_NAME}_INCLUDE_DIR "\${${_PROJECT_NAME}_CMAKE_DIR}/${REL_INCLUDE_DIR}")
set(CONFIG_SOURCE_DIR)
set(CONFIG_DIR)
configure_file(
        ${CMAKE_CURRENT_SOURCE_DIR}/${PROJECT_NAME}Config.cmake.in
        ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake
        @ONLY)

install(FILES "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
        DESTINATION ${CMAKECONFIG_INSTALL_DIR})
if (CMAKE_INSTALL_DIR)
    install(FILES
            ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake
            ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake
            DESTINATION "${CMAKE_INSTALL_DIR}")
endif ()

# uninstall target
if (NOT TARGET uninstall)
    configure_file(
            "${CMAKE_CURRENT_SOURCE_DIR}/${PROJECT_NAME}Uninstall.cmake.in"
            "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Uninstall.cmake"
            IMMEDIATE @ONLY)
    add_custom_target(uninstall
            COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Uninstall.cmake)
endif ()

message("The directory the headers are installed in: ${CMAKE_INSTALL_PREFIX}")
message("Directory where lib will install: ${LIB_INSTALL_DIR}")
message("")
