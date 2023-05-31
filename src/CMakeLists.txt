# core: used by all
file(GLOB CORE_SRCS
        lzio.c
        lctype.c
        lopcodes.c
        lmem.c
        lundump.c
        ldump.c
        lstate.c
        lgc.c
        llex.c
        lcode.c
        lparser.c
        ldebug.c
        lfunc.c
        lobject.c
        ltm.c
        lstring.c
        ltable.c
        ldo.c
        lvm.c
        lapi.c
        )

# auxiliary library: used by all
file(GLOB AUX_SRCS
        lauxlib.c
        )

# standard library: not used by luac
file(GLOB LIB_SRCS
        lbaselib.c
        lcorolib.c
        ldblib.c
        liolib.c
        lmathlib.c
        loadlib.c
        loslib.c
        lstrlib.c
        ltablib.c
        lutf8lib.c
        linit.c
        )

# lua
file(GLOB LUA_SRCS
        lua.c
        )

add_library(lua_include INTERFACE)
target_include_directories(lua_include INTERFACE
        $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:include>
        )
add_library(lua_lib STATIC ${CORE_SRCS} ${AUX_SRCS} ${LIB_SRCS})
target_link_libraries(lua_lib PUBLIC lua_include)
if (UNIX)
    if (APPLE)
        set(_LUA_PLATFORM LUA_USE_MACOSX)
    else ()
        set(_LUA_PLATFORM LUA_USE_LINUX)
    endif ()
    target_compile_definitions(lua_lib INTERFACE ${_LUA_PLATFORM})
    target_link_libraries(lua_lib PUBLIC dl)
endif ()
set_target_properties(lua_lib PROPERTIES
        VERSION "${PACKAGE_VERSION}"
        OUTPUT_NAME lua
        )

add_library(Lua::Library ALIAS lua_lib)

add_executable(lua ${LUA_SRCS})
target_link_libraries(lua PUBLIC lua_include PRIVATE lua_lib)
include(CheckIncludeFile)
CHECK_INCLUDE_FILE("readline/readline.h" HAVE_READLINE)
if (HAVE_READLINE)
    target_compile_definitions(lua PRIVATE LUA_USE_READLINE)
    target_link_libraries(lua PUBLIC readline)
endif ()

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
    configure_file(${CMAKE_CURRENT_SOURCE_DIR}/${PROJECT_NAME}.pc.in
            ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.pc
            @ONLY)
    install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.pc
            DESTINATION "${LIB_INSTALL_DIR}/pkgconfig"
            COMPONENT pkgconfig)
endif ()

install(TARGETS lua_include lua_lib
        EXPORT LuaTargets
        )

install(DIRECTORY include/
        DESTINATION "${INCLUDE_INSTALL_DIR}"
        )

install(EXPORT LuaTargets
        FILE LuaTargets.cmake
        DESTINATION ${CMAKE_INSTALL_DIR}
        NAMESPACE Lua::
        )

string(TOUPPER ${PROJECT_NAME} _PROJECT_NAME)

set(CONFIG_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR})
set(CONFIG_DIR ${CMAKE_CURRENT_BINARY_DIR})
set(${_PROJECT_NAME}_INCLUDE_DIR "\${${_PROJECT_NAME}_SOURCE_DIR}/include")
get_target_property(LUA_EXPORT_LIBRARY Lua::Library ALIASED_TARGET)
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
        DESTINATION ${CMAKECONFIG_INSTALL_DIR}
        )
if (CMAKE_INSTALL_DIR)
    install(FILES
            ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake
            ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake
            DESTINATION "${CMAKE_INSTALL_DIR}"
            )
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
