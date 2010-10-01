# Copyright (C) 2007-2009 LuaDist.
# Contributions from David Manura, Robert G. Jakabosky.
# Redistribution and use of this file is allowed according to the terms of the MIT license.
# For details see the COPYRIGHT file distributed with LuaDist.
# Please note that the package source code is licensed under its own license.

PROJECT ( luajit2 C )
CMAKE_MINIMUM_REQUIRED(VERSION 2.8)
INCLUDE(dist.cmake)

include(CheckCCompilerFlag)

# LuaJIT version (for packaging and deployment)
set(LUA_JIT_VERSION_MAJOR 2)
set(LUA_JIT_VERSION_MINOR 0)
set(LUA_JIT_VERSION_PATCH 0)
set(LUA_JIT_VERSION_TAG "-beta2")
set(LUA_JIT_VERSION
	"${LUA_JIT_VERSION_MAJOR}.${LUA_JIT_VERSION_MINOR}.${LUA_JIT_VERSION_PATCH}")
set(LUA_JIT_SOVERSION
	"${LUA_JIT_VERSION_MAJOR}.${LUA_JIT_VERSION_MINOR}")

# LuaJIT package info.
set(CPACK_PACKAGE_VERSION_MAJOR ${LUA_JIT_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR ${LUA_JIT_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH ${LUA_JIT_VERSION_PATCH})
#set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_SOURCE_DIR}/COPYRIGHT")
set(CPACK_RESOURCE_FILE_README "${CMAKE_SOURCE_DIR}/README")
set(CPACK_PACKAGE_VENDOR "Mike Pall.")
set(CPACK_SOURCE_GENERATOR "TGZ")
set(CPACK_SOURCE_IGNORE_FILES
"/\\\\.;/\\\\.git.*/;~$;build/;CMakeFiles/;CMakeCache;Testing/;cmake_install;CPack;Dart;Makefile$")
set(CPACK_SOURCE_PACKAGE_FILE_NAME
	"${CMAKE_PROJECT_NAME}-${LUA_JIT_VERSION}${LUA_JIT_VERSION_TAG}")
# MUST be after CPACK_* variables.
include(CPack)


INCLUDE_DIRECTORIES(dynasm src)
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_BINARY_DIR})

IF ( WIN32 AND NOT CYGWIN)
    ADD_DEFINITIONS ( -DLUA_BUILD_AS_DLL )
    IF(MSVC)
        ADD_DEFINITIONS(-D_CRT_SECURE_NO_DEPRECATE )
        #~2DO? SET(DEF_FILE src/lua.def)
        #~SET(DLL_RC_FILE src/lua_dll.rc)
        #~SET(LUA_RC_FILE src/lua.rc)
        #~SET(LUAC_RC_FILE src/lua_simple.rc)
    ENDIF()
ELSE ( )
    # redundant in LuaJIT: ADD_DEFINITIONS ( -DLUA_USE_POSIX -DLUA_USE_DLOPEN )
    SET ( LIBS m dl )
ENDIF ( )

option(WANT_SHARED_LIBRARY "Set to ON to build dynamic library." ON)
option(WANT_STATIC_LIBRARY "Set to ON to build static library." ON)

# Apple specific tweaks
IF (APPLE)
       MESSAGE("LuaJIT only supports i386 architecture type.")
       MESSAGE("Make sure to compile with CMAKE_OSX_ARCHITECTURES set to i386 or ppc on 64bit systems")
       MESSAGE("Additionally set CMAKE_ASM_FLAGS='-arch i386'.")
ENDIF()

###
# Compile time options.  Descriptions taken from Makefile.
# Enable these as needed.
# 
# Note that most of these are NOT suitable for benchmarking or release mode!
#(XCFLAGS)

# Disable the use of CMOV and FCOMI*/FUCOMI* instructions in the interpreter.
# This is only necessary if you intend to run the code on REALLY ANCIENT CPUs
# (before Pentium Pro, or on the VIA C3). This generally slows down the
# interpreter. Don't bother if your OS wouldn't run on them, anyway.
option(LUAJIT_CPU_NOCMOV "Disable the use of CMOV and FCOMI*/FUCOMI* instructions." OFF)
if(LUAJIT_CPU_NOCMOV)
  add_definitions(-DLUAJIT_CPU_NOCMOV)
endif(LUAJIT_CPU_NOCMOV)

# Disable the JIT compiler, i.e. turn LuaJIT into a pure interpreter:
option(LUAJIT_USE_DISABLE_JIT "Disable the JIT compiler." OFF)
if(LUAJIT_USE_DISABLE_JIT)
  add_definitions(-DLUAJIT_DISABLE_JIT)
endif()

# Use the system provided memory allocator (realloc) instead of the
# bundled memory allocator. This is slower, but sometimes helpful for
# debugging. It's mandatory for Valgrind's memcheck tool, too.
option(LUAJIT_USE_SYSMALLOC "Use the system provided memory allocator." OFF)
if(LUAJIT_USE_SYSMALLOC)
  add_definitions(-DLUAJIT_USE_SYSMALLOC)
endif()

# This define is required to run LuaJIT under Valgrind. The Valgrind
# header files must be installed. You should enable debug information, too.
option(LUAJIT_USE_VALGRIND "This define is required to run LuaJIT under Valgrind." OFF)
if(LUAJIT_USE_VALGRIND)
  add_definitions(-DLUAJIT_USE_VALGRIND)
endif()

# This is the client for the GDB JIT API. GDB 7.0 or higher is required
# to make use of it. See lj_gdbjit.c for details. Enabling this causes
# a non-negligible overhead, even when not running under GDB.
option(LUAJIT_USE_GDBJIT "This is the client for the GDB JIT API." OFF)
if(LUAJIT_USE_GDBJIT)
  add_definitions(-DLUAJIT_USE_GDBJIT)
endif()

# Turn on assertions for the Lua/C API to debug problems with lua_* calls.
# This is rather slow -- use only while developing C libraries/embeddings.
option(LUA_USE_APICHECK "Enable API checks." OFF)
if(LUA_USE_APICHECK)
  add_definitions(-DLUA_USE_APICHECK)
endif()

# Turn on assertions for the whole LuaJIT VM. This significantly slows down
# everything. Use only if you suspect a problem with LuaJIT itself.
option(LUA_USE_ASSERT "Enable assertions." OFF)
if(LUA_USE_ASSERT)
  add_definitions(-DLUA_USE_ASSERT)
endif()


check_c_compiler_flag("-fno-stack-protector" HAS_STACK_PROTECTOR)
if(HAS_STACK_PROTECTOR)
  add_definitions("-fno-stack-protector")
endif(HAS_STACK_PROTECTOR)



###


#~2DO: LuaJIT currently doesn't support readline/curses (which is
# planned to be moved into a module loaded only in interactive mode).


# Optionally recompile buildvm_x86.h.
IF (NOT EXISTS src/buildvm_x86.h)  #~ improve?
    FIND_PROGRAM(LUA NAMES lua lua.bat)
    SET(DASM ${LUA} "${CMAKE_CURRENT_SOURCE_DIR}/dynasm/dynasm.lua")
    ADD_CUSTOM_COMMAND(
        OUTPUT ${CMAKE_CURRENT_SOURCE_DIR}/src/buildvm_x86.h
        DEPENDS src/buildvm_x86.dasc
        COMMAND ${DASM} "-LN" "-o" buildvm_x86.h buildvm_x86.dasc
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/src )
ENDIF()

ADD_EXECUTABLE(buildvm src/buildvm.c src/buildvm_fold.c src/buildvm_peobj.c src/buildvm_asm.c src/buildvm_lib.c src/buildvm_x86.h)

SET(LJLIB_SRC
  ${CMAKE_CURRENT_SOURCE_DIR}/src/lib_base.c
  ${CMAKE_CURRENT_SOURCE_DIR}/src/lib_math.c
  ${CMAKE_CURRENT_SOURCE_DIR}/src/lib_bit.c
  ${CMAKE_CURRENT_SOURCE_DIR}/src/lib_string.c
  ${CMAKE_CURRENT_SOURCE_DIR}/src/lib_table.c
  ${CMAKE_CURRENT_SOURCE_DIR}/src/lib_io.c
  ${CMAKE_CURRENT_SOURCE_DIR}/src/lib_os.c
  ${CMAKE_CURRENT_SOURCE_DIR}/src/lib_package.c
  ${CMAKE_CURRENT_SOURCE_DIR}/src/lib_debug.c
  ${CMAKE_CURRENT_SOURCE_DIR}/src/lib_jit.c )

SET(BUILDVM ${CMAKE_CURRENT_BINARY_DIR}/buildvm)

macro(add_buildvm_target _target _mode)
    add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${_target}
        COMMAND ${BUILDVM} ARGS -m ${_mode} -o ${CMAKE_CURRENT_BINARY_DIR}/${_target} ${ARGN}
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        DEPENDS ${BUILDVM} ${ARGN}
    )
endmacro(add_buildvm_target)

IF (WIN32) # PE for any Windows, including Cygwin
    add_buildvm_target(lj_vm.obj peobj)
    SET(LJVM_SRC ${CMAKE_CURRENT_BINARY_DIR}/lj_vm.obj)
ELSE()
    add_buildvm_target(lj_vm.s asm)
    SET(LJVM_SRC ${CMAKE_CURRENT_BINARY_DIR}/lj_vm.s)
    ENABLE_LANGUAGE(ASM)
ENDIF()
add_buildvm_target(lj_ffdef.h   ffdef   ${LJLIB_SRC})
add_buildvm_target(lj_libdef.h  libdef  ${LJLIB_SRC})
add_buildvm_target(lj_recdef.h  recdef  ${LJLIB_SRC})
add_buildvm_target(vmdef.lua    vmdef   ${LJLIB_SRC})
add_buildvm_target(lj_folddef.h folddef src/lj_opt_fold.c)
ADD_CUSTOM_TARGET(vmdef ALL DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/vmdef.lua")

SET(LJCORE_SRC
    src/lj_gc.c src/lj_err.c src/lj_ctype.c src/lj_bc.c src/lj_obj.c
    src/lj_str.c src/lj_tab.c src/lj_func.c src/lj_udata.c src/lj_meta.c
    src/lj_state.c src/lj_dispatch.c src/lj_vmevent.c src/lj_api.c
    src/lj_lex.c src/lj_parse.c
    src/lj_ir.c src/lj_opt_mem.c src/lj_opt_fold.c src/lj_opt_narrow.c
    src/lj_opt_dce.c src/lj_opt_loop.c
    src/lj_mcode.c src/lj_snap.c src/lj_record.c src/lj_asm.c
                   src/lj_trace.c src/lj_gdbjit.c
    src/lj_lib.c src/lj_alloc.c src/lib_aux.c
    ${LJLIB_SRC}
    src/lib_init.c )

SET(LJVMCORE_SRC ${LJVM_SRC} ${LJCORE_SRC})

SET(LJVMCORE_DEPS ${CMAKE_CURRENT_BINARY_DIR}/lj_ffdef.h
                  ${CMAKE_CURRENT_BINARY_DIR}/lj_libdef.h
                  ${CMAKE_CURRENT_BINARY_DIR}/lj_recdef.h
                  ${CMAKE_CURRENT_BINARY_DIR}/lj_folddef.h )

SET(LUA_JIT_LIBS)

if(WANT_STATIC_LIBRARY)
  ADD_LIBRARY(liblua_static STATIC ${LJVMCORE_SRC} ${LJVMCORE_DEPS})
  SET_TARGET_PROPERTIES ( liblua_static PROPERTIES OUTPUT_NAME lua CLEAN_DIRECT_OUTPUT 1 )
  TARGET_LINK_LIBRARIES ( liblua_static ${LIBS})
  set(LUA_JIT_LIBS ${LUA_JIT_LIBS} liblua_static)
  set(LUA_JIT_MAIN_LIB liblua_static)
endif(WANT_STATIC_LIBRARY)

if(WANT_SHARED_LIBRARY)
  ADD_LIBRARY(liblua_shared SHARED ${LJVMCORE_SRC} ${LJVMCORE_DEPS})
  SET_TARGET_PROPERTIES ( liblua_shared PROPERTIES OUTPUT_NAME lua CLEAN_DIRECT_OUTPUT 1 )
  TARGET_LINK_LIBRARIES ( liblua_shared ${LIBS})
  set(LUA_JIT_LIBS ${LUA_JIT_LIBS} liblua_shared)
  set(LUA_JIT_MAIN_LIB liblua_shared)
endif(WANT_SHARED_LIBRARY)

#~2DO: Windows icon .ico .rc ?
#~2DO: what to do about luac? (not available)
#~2DO: install regular Lua docs?

#~2DO
# :AMALGDLL
# %LJCOMPILE% /DLUA_BUILD_AS_DLL ljamalg.c
# %LJLINK% /DLL /out:lua51.dll ljamalg.obj lj_vm.obj

ADD_EXECUTABLE(lua src/luajit.c)
TARGET_LINK_LIBRARIES(lua ${LUA_JIT_MAIN_LIB})

# Install
INSTALL ( TARGETS lua RUNTIME DESTINATION ${INSTALL_BIN} ) #~ luac
INSTALL ( TARGETS ${LUA_JIT_LIBS} DESTINATION ${INSTALL_LIB} )
INSTALL ( FILES src/lua.h src/luaconf.h src/lualib.h src/lauxlib.h
          DESTINATION ${INSTALL_INC} ) #~ etc/lua.hpp
INSTALL ( FILES lib/bc.lua lib/dis_x64.lua lib/dis_x86.lua lib/dump.lua
                lib/v.lua ${CMAKE_CURRENT_BINARY_DIR}/vmdef.lua
                DESTINATION ${INSTALL_LMOD}/jit )
INSTALL ( FILES etc/strict.lua DESTINATION ${INSTALL_LMOD} )
INSTALL ( FILES src/lua.hpp DESTINATION ${INSTALL_INC} )
INSTALL ( DIRECTORY doc etc DESTINATION ${INSTALL_DATA} PATTERN ".svn" EXCLUDE )
INSTALL ( FILES README COPYRIGHT DESTINATION ${INSTALL_DATA} )
INSTALL ( FILES etc/luajit.1 etc/luajit.pc DESTINATION ${INSTALL_FOO} )

