# Copyright (C) 2007-2013 LuaDist.
# Created by Peter Drahoš
# Redistribution and use of this file is allowed according to the terms of the MIT license.
# For details see the COPYRIGHT file distributed with LuaDist.
# Please note that the package source code is licensed under its own license.

# NOTE: This build is currently only supporting x86 targets, for other targets use the original makefile. Please do not submit bugs to the LuaJIT author in case this build fails, instead use http://github.com/LuaDist/luajit

project ( luajit C ASM)
cmake_minimum_required ( VERSION 2.8 )
include ( cmake/dist.cmake )
include ( lua )

## CONFIGURATION
# Default configuration (we assume POSIX by default)
set ( LUA_PATH "LUA_PATH" CACHE STRING "Environment variable to use as package.path." )
set ( LUA_CPATH "LUA_CPATH" CACHE STRING "Environment variable to use as package.cpath." )
set ( LUA_INIT "LUA_INIT" CACHE STRING "Environment variable for initial script." )

option ( LUA_ANSI "Use only ansi features." OFF )
option ( LUA_USE_RELATIVE_LOADLIB "Use modified loadlib.c with support for relative paths on posix systems." ON)
set ( LUA_IDSIZE 60 CACHE NUMBER "gives the maximum size for the description of the source." )
set ( LUA_PROMPT "> " CACHE STRING "Is the default prompt used by stand-alone Lua." )
set ( LUA_PROMPT2 ">> " CACHE STRING "Is the default continuation prompt used by stand-alone Lua." )
set ( LUA_MAXINPUT 512 CACHE NUMBER "Is the maximum length for an input line in the stand-alone interpreter.")

# Version
set ( MAJVER 2 )
set ( MINVER 0 )
set ( RELVER 2 )
set ( ABIVER 5.1 )
set ( NODOTABIVER 51 )

# Extra flags
option ( LUAJIT_DISABLE_FFI "Permanently disable the FFI extension to reduce the size of the LuaJIT executable. But please consider that the FFI library is compiled-in, but NOT loaded by default. It only allocates any memory, if you actually make use of it." OFF )
option ( LUAJIT_ENABLE_LUA52COMPAT "Features from Lua 5.2 that are unlikely to break existing code are enabled by default. Some other features that *might* break some existing code (e.g. __pairs or os.execute() return values) can be enabled here. Note: this does not provide full compatibility with Lua 5.2 at this time." ON )
option ( LUAJIT_DISABLE_JIT "Disable the JIT compiler, i.e. turn LuaJIT into a pure interpreter." OFF )

option ( LUAJIT_USE_SYSMALLOC "Use the system provided memory allocator (realloc) instead of the bundled memory allocator. This is slower, but sometimes helpful for debugging. It's helpful for Valgrind's memcheck tool, too. This option cannot be enabled on x64, since the built-in allocator is mandatory." OFF )
option ( LUAJIT_USE_VALGRIND "This option is required to run LuaJIT under Valgrind. The Valgrind header files must be installed. You should enable debug information, too." OFF )
option ( LUAJIT_USE_GDBJIT "This is the client for the GDB JIT API. GDB 7.0 or higher is required to make use of it. See lj_gdbjit.c for details. Enabling this causes a non-negligible overhead, even when not running under GDB." OFF )

option ( LUA_USE_APICHECK "Turn on assertions for the Lua/C API to debug problems with lua_* calls. This is rather slow, use only while developing C libraries/embeddings." OFF )
option ( LUA_USE_ASSERT "Turn on assertions for the whole LuaJIT VM. This significantly slows down everything. Use only if you suspect a problem with LuaJIT itself." OFF )

option ( LUAJIT_CPU_SSE2 "Disable SSE2." OFF )
option ( LUAJIT_CPU_NOCMOV "Disable NOCMOV." OFF )

# Tunable variables
set ( LUAI_MAXSTACK 65500 CACHE NUMBER "Max. # of stack slots for a thread (<64K)." )
set ( LUAI_MAXCSTACK 8000 CACHE NUMBER "Max. # of stack slots for a C func (<10K)." )
set ( LUAI_GCPAUSE 200 CACHE NUMBER "Pause GC until memory is at 200%." )
set ( LUAI_GCMUL 200 CACHE NUMBER "Run GC at 200% of allocation speed." )
set ( LUA_MAXCAPTURES 32 CACHE NUMBER "Max. pattern captures." )

## SETUP
set ( LUA_DIRSEP "/" )
set ( LUA_MODULE_SUFFIX ${CMAKE_SHARED_MODULE_SUFFIX} )
set ( LUA_LDIR ${INSTALL_LMOD} )
set ( LUA_CDIR ${INSTALL_CMOD} )

if ( LUA_USE_RELATIVE_LOADLIB )
  # This will set up relative paths to lib
  string ( REGEX REPLACE "[^!/]+" ".." LUA_DIR "!/${INSTALL_BIN}/" )
else ()
  # Direct path to installation
  set ( LUA_DIR ${CMAKE_INSTALL_PREFIX} CACHE STRING "Destination from which modules will be resolved. See INSTALL_LMOD and INSTALL_CMOD.")
endif ()

set ( LUA_PATH_DEFAULT "./?.lua;${LUA_DIR}${LUA_LDIR}/?.lua;${LUA_DIR}${LUA_LDIR}/?/init.lua;./?/init.lua" )
set ( LUA_CPATH_DEFAULT "./?${LUA_MODULE_SUFFIX};${LUA_DIR}${LUA_CDIR}/?${LUA_MODULE_SUFFIX};${LUA_DIR}${LUA_CDIR}/loadall${LUA_MODULE_SUFFIX}" )

if ( WIN32 AND NOT CYGWIN )
  # Windows systems
  add_definitions ( -DLUA_BUILD_AS_DLL -DLUAJIT_OS=LUAJIT_OS_WINDOWS)
  option ( LUA_BUILD_WLUA "Build wluajit interpreter for no-console applications." ON )
  set ( LJVM_MODE peobj )
  # Paths (Double escapes needed)
  set ( LUA_DIRSEP "\\\\" )
  string ( REPLACE "/" ${LUA_DIRSEP} LUA_DIR "${LUA_DIR}" )
  string ( REPLACE "/" ${LUA_DIRSEP} LUA_LDIR "${LUA_LDIR}" )
  string ( REPLACE "/" ${LUA_DIRSEP} LUA_CDIR "${LUA_CDIR}" )
  string ( REPLACE "/" ${LUA_DIRSEP} LUA_PATH_DEFAULT "${LUA_PATH_DEFAULT}" )
  string ( REPLACE "/" ${LUA_DIRSEP} LUA_CPATH_DEFAULT "${LUA_CPATH_DEFAULT}" )

elseif ( APPLE )
  set ( CMAKE_EXE_LINKER_FLAGS "-pagezero_size 10000 -image_base 100000000 -image_base 7fff04c4a000" )
  option ( LUA_USE_POSIX "Use POSIX functionality." ON )
  option ( LUA_USE_DLOPEN "Use dynamic linker to load modules." ON )
  set ( LJVM_MODE machasm )
else ()
  option ( LUA_USE_POSIX "Use POSIX functionality." ON )
  option ( LUA_USE_DLOPEN "Use dynamic linker to load modules." ON )
  set ( LJVM_MODE elfasm )
endif ()

## LIBRARY DETECTION
# Optional libraries
find_package ( Readline )
if ( READLINE_FOUND )
  option ( LUA_USE_READLINE "Use readline in the Lua CLI." ON )
endif ()

find_package ( Curses )
if ( CURSES_FOUND )
  option ( LUA_USE_CURSES "Use curses in the Lua CLI." ON )
endif ()

# Setup needed variables and libraries
if ( LUA_USE_POSIX )
  # On POSIX Lua links to standard math library "m"
  list ( APPEND LIBS m )
endif ()

if ( LUA_USE_DLOPEN )
  # Link to dynamic linker library "dl"
  list ( APPEND LIBS dl )
endif ()

if ( LUA_USE_READLINE )
  # Add readline
  include_directories ( ${READLINE_INCLUDE_DIR} )
  list ( APPEND LIBS ${READLINE_LIBRARY} )
endif ()

if ( LUA_USE_CURSES )
  # Add curses
  include_directories ( ${CURSES_INCLUDE_DIR} )
  list ( APPEND LIBS ${CURSES_LIBRARY} )
endif ()

## SOURCES
# Generate luaconf.h
configure_file ( src/luaconf.h.in ${CMAKE_CURRENT_BINARY_DIR}/luaconf.h )

set ( LJLIB_C
  src/lib_base.c
  src/lib_math.c
  src/lib_bit.c
  src/lib_string.c
  src/lib_table.c
  src/lib_io.c
  src/lib_os.c
  src/lib_debug.c
  src/lib_jit.c
  src/lib_ffi.c
)
if ( LUA_USE_RELATIVE_LOADLIB )
  list ( APPEND LJLIB_C src/lib_package_rel.c )
else ()
  list ( APPEND LJLIB_C src/lib_package.c )
endif ()

## GENERATE VM
# Build minilua
add_executable ( minilua src/host/minilua.c )
target_link_libraries ( minilua ${LIBS} )

# Dynasm
set ( DASM ${CMAKE_CURRENT_SOURCE_DIR}/dynasm/dynasm.lua )
set ( DASM_T ${CMAKE_CURRENT_SOURCE_DIR}/src/host/buildvm_arch.h )

# 2DO: Proper detection of flags
set ( DASM_VER "" )
set ( DASM_FLAGS -D FPU -D HFABI )
set ( DASM_ARCH x86 )

# Raspberry PI, ARM
if ( ${CMAKE_SYSTEM_PROCESSOR} MATCHES "armv6l" )
  set ( DASM_ARCH arm )
  list ( APPEND DASM_FLAGS -D DUALNUM )
  set ( DASM_VER 60 )
endif ()

# Windows is ... special
if ( WIN32 )
  list ( APPEND DASM_FLAGS -D WIN )
endif ()

# 32bit vs 64bit
if ( CMAKE_SIZEOF_VOID_P EQUAL 8 )
  list ( APPEND DASM_FLAGS -D P64 )
endif ()

if ( NOT LUAJIT_DISABLE_JIT )
  list ( APPEND DASM_FLAGS -D JIT )
endif ()

if ( NOT LUAJIT_DISABLE_FFI )
  list ( APPEND DASM_FLAGS -D FFI )
endif ()

if ( NOT LUAJIT_CPU_SSE2 )
  list ( APPEND DASM_FLAGS -D SSE2 )
endif ()

list ( APPEND DASM_FLAGS -D VER=${DASM_VER} )

string ( REPLACE ";" " " DASM_FLAGS_STR "${DASM_FLAGS}")

message ( "DASM_FLAGS: ${DASM_FLAGS_STR}")
message ( "DASM_ARCH: ${DASM_ARCH}" )

set ( DASM_DASC ${CMAKE_CURRENT_SOURCE_DIR}/src/vm_${DASM_ARCH}.dasc )

# Generate buildvm arch header
add_custom_command(OUTPUT ${DASM_T}
  COMMAND minilua ${DASM} ${DASM_FLAGS} -o ${DASM_T} ${DASM_DASC}
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/dynasm
  DEPENDS minilua
)

# Buildvm
file ( GLOB SRC_BUILDVM src/host/buildvm*.c )
add_executable ( buildvm ${SRC_BUILDVM} ${DASM_T} )

macro(add_buildvm_target _target _mode)
  add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${_target}
    COMMAND buildvm ARGS -m ${_mode} -o ${CMAKE_CURRENT_BINARY_DIR}/${_target} ${ARGN}
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    DEPENDS buildvm ${ARGN}
  )
endmacro(add_buildvm_target)

if ( WIN32 AND NOT CYGWIN )
  add_buildvm_target ( lj_vm.obj peobj )
  set (LJ_VM_SRC ${CMAKE_CURRENT_BINARY_DIR}/lj_vm.obj)
else ()
  add_buildvm_target ( lj_vm.s ${LJVM_MODE} )
  set (LJ_VM_SRC ${CMAKE_CURRENT_BINARY_DIR}/lj_vm.s)
endif ()

add_buildvm_target ( lj_ffdef.h   ffdef   ${LJLIB_C} )
add_buildvm_target ( lj_bcdef.h  bcdef  ${LJLIB_C} )
add_buildvm_target ( lj_folddef.h folddef src/lj_opt_fold.c )
add_buildvm_target ( lj_recdef.h  recdef  ${LJLIB_C} )
add_buildvm_target ( lj_libdef.h  libdef  ${LJLIB_C} )
add_buildvm_target ( jit/vmdef.lua  libvm  ${LJLIB_C} )

SET ( DEPS
  ${LJ_VM_SRC}
  ${CMAKE_CURRENT_BINARY_DIR}/lj_ffdef.h
  ${CMAKE_CURRENT_BINARY_DIR}/lj_bcdef.h
  ${CMAKE_CURRENT_BINARY_DIR}/lj_libdef.h
  ${CMAKE_CURRENT_BINARY_DIR}/lj_recdef.h
  ${CMAKE_CURRENT_BINARY_DIR}/lj_folddef.h
)

## LuaJIT Library
include_directories ( ${CMAKE_CURRENT_BINARY_DIR} dynasm src src/host )

set ( LJCORE_C
  src/lj_gc.c
  src/lj_err.c
  src/lj_char.c
  src/lj_bc.c
  src/lj_obj.c
  src/lj_str.c
  src/lj_tab.c
  src/lj_func.c
  src/lj_udata.c
  src/lj_meta.c
  src/lj_debug.c
  src/lj_state.c
  src/lj_dispatch.c
  src/lj_vmevent.c
  src/lj_vmmath.c
  src/lj_strscan.c
  src/lj_api.c
  src/lj_lex.c
  src/lj_parse.c
  src/lj_bcread.c
  src/lj_bcwrite.c
  src/lj_load.c
  src/lj_ir.c
  src/lj_opt_mem.c
  src/lj_opt_fold.c
  src/lj_opt_narrow.c
  src/lj_opt_dce.c
  src/lj_opt_loop.c
  src/lj_opt_split.c
  src/lj_opt_sink.c
  src/lj_mcode.c
  src/lj_snap.c
  src/lj_record.c
  src/lj_crecord.c
  src/lj_ffrecord.c
  src/lj_asm.c
  src/lj_trace.c
  src/lj_gdbjit.c
  src/lj_ctype.c
  src/lj_cdata.c
  src/lj_cconv.c
  src/lj_ccall.c
  src/lj_ccallback.c
  src/lj_carith.c
  src/lj_clib.c
  src/lj_cparse.c
  src/lj_lib.c
  src/lj_alloc.c
  src/lib_aux.c
  src/lib_init.c
  ${LJLIB_C}
)

add_library ( liblua SHARED ${LJCORE_C} ${DEPS} )
target_link_libraries ( liblua ${LIBS} )
set_target_properties ( liblua PROPERTIES OUTPUT_NAME lua CLEAN_DIRECT_OUTPUT 1 )

## LuaJIT Executable
add_executable ( luajit src/luajit.c src/luajit.rc )
target_link_libraries ( luajit liblua )

# On Windows build a no-console variant also
if ( LUA_BUILD_WLUA )
  add_executable ( wluajit WIN32 src/wmain.c src/luajit.c src/luajit.rc )
  target_link_libraries ( wluajit liblua )
  install_executable ( wluajit )
endif ()

## INSTALL
install_executable( luajit )
install_library( liblua )
install_header( src/lua.h src/luajit.h src/lua.hpp ${CMAKE_CURRENT_BINARY_DIR}/luaconf.h src/lualib.h src/lauxlib.h )

install_doc ( doc/ )
install_foo ( etc/ )
install_data ( COPYRIGHT README.md )
