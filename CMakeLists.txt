cmake_minimum_required(VERSION 3.27)

project(lua
    VERSION 5.4.8
    LANGUAGES C CXX
)

option(LUA_BUILD_BINARY "Build the lua interpreter executable" ON)
option(LUA_BUILD_AS_CXX "Compile Lua source as C++" OFF)

find_library(MATH_LIBRARY m)

set(LUA_CORE_SRC
    lapi.c lcode.c lctype.c ldebug.c ldo.c ldump.c 
    lfunc.c lgc.c llex.c lmem.c lobject.c lopcodes.c 
    lparser.c lstate.c lstring.c ltable.c ltm.c 
    lundump.c lvm.c lzio.c ltests.c
)

set(LUA_AUX_SRC
    lauxlib.c
)

set(LUA_LIB_SRC
    lbaselib.c ldblib.c liolib.c lmathlib.c loslib.c 
    ltablib.c lstrlib.c lutf8lib.c loadlib.c lcorolib.c 
    linit.c
)

add_library(liblua ${LUA_CORE_SRC} ${LUA_AUX_SRC} ${LUA_LIB_SRC})
add_library(lua::lib ALIAS liblua)

if(LUA_BUILD_AS_CXX)
    set_source_files_properties(${LUA_CORE_SRC} ${LUA_AUX_SRC} ${LUA_LIB_SRC} PROPERTIES LANGUAGE CXX)
else()
    set_target_properties(liblua PROPERTIES C_STANDARD 99)
endif()

target_include_directories(liblua PUBLIC 
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    $<INSTALL_INTERFACE:include>
)

if(MATH_LIBRARY)
    target_link_libraries(liblua PUBLIC ${MATH_LIBRARY})
endif()

if(UNIX)
    target_link_libraries(liblua PUBLIC ${CMAKE_DL_LIBS})
endif()

if(APPLE)
    target_compile_definitions(liblua PUBLIC LUA_USE_MACOSX)
elseif(UNIX)
    target_compile_definitions(liblua PUBLIC LUA_USE_LINUX)
elseif(WIN32)
    target_compile_definitions(liblua PUBLIC LUA_USE_WINDOWS)
    target_compile_definitions(liblua PUBLIC LUA_BUILD_AS_DLL)
endif()

if(MSVC)
    target_compile_options(liblua PRIVATE /W3)
else()
    target_compile_options(liblua PRIVATE -Wall -O2 -fno-stack-protector -fno-common)
endif()

if(LUA_BUILD_BINARY)
    add_executable(lua_interpreter lua.c)
    add_executable(lua::interpreter ALIAS lua_interpreter)
    
    set_target_properties(lua_interpreter PROPERTIES OUTPUT_NAME lua)
    target_link_libraries(lua_interpreter PRIVATE liblua)
    
    if(MATH_LIBRARY)
        target_link_libraries(lua_interpreter PRIVATE ${MATH_LIBRARY})
    endif()

    if(UNIX AND NOT APPLE)
        target_link_options(lua_interpreter PRIVATE -Wl,-E)
    endif()
endif()

include(GNUInstallDirs)

install(TARGETS liblua
    EXPORT luaTargets
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

if(LUA_BUILD_BINARY)
    install(TARGETS lua_interpreter
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    )
endif()

install(FILES 
    lua.h luaconf.h lualib.h lauxlib.h lua.hpp
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)