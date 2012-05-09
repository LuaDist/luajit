# Copyright (C) 2007-2011 LuaDist.
# Created by Peter Drahoš
# Redistribution and use of this file is allowed according to the terms of the MIT license.
# For details see the COPYRIGHT file distributed with LuaDist.
# Please note that the package source code is licensed under its own license.

project ( luajit C ASM)
cmake_minimum_required ( VERSION 2.6 )
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

# LuaJIT specific
option ( LUAJIT_DISABLE_FFI "Disable FFI." OFF )
option ( LUAJIT_ENABLE_LUA52COMPAT "Enable Lua 5.2 compatibility." ON )
option ( LUAJIT_DISABLE_JIT "Disable JIT." OFF )
option ( LUAJIT_CPU_SSE2 "Disable SSE2." OFF )
option ( LUAJIT_CPU_NOCMOV "Disable NOCMOV." OFF )
option ( LUAJIT_RUN_DYNASM "Re-generate headers with DynASM." OFF )

#2DO: LUAI_* and LUAL_* settings, for now defaults are used.
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
  set ( LJVM_MODE coffasm )
  # Paths (Double escapes needed)
  set ( LUA_DIRSEP "\\\\" )
  string ( REPLACE "/" ${LUA_DIRSEP} LUA_DIR "${LUA_DIR}" )
  string ( REPLACE "/" ${LUA_DIRSEP} LUA_LDIR "${LUA_LDIR}" )
  string ( REPLACE "/" ${LUA_DIRSEP} LUA_CDIR "${LUA_CDIR}" )
  string ( REPLACE "/" ${LUA_DIRSEP} LUA_PATH_DEFAULT "${LUA_PATH_DEFAULT}" )
  string ( REPLACE "/" ${LUA_DIRSEP} LUA_CPATH_DEFAULT "${LUA_CPATH_DEFAULT}" )

elseif ( APPLE )
  set ( CMAKE_EXE_LINKER_FLAGS "-pagezero_size 10000 -image_base 100000000" )
  option ( LUA_USE_POSIX "Use POSIX functionality." ON )
  option ( LUA_USE_DLOPEN "Use dynamic linker to load modules." ON )
  set ( LJVM_MODE machasm )
else ()
  option ( LUA_USE_POSIX "Use POSIX functionality." ON )
  option ( LUA_USE_DLOPEN "Use dynamic linker to load modules." ON )
  set ( LJVM_MODE elfasm )  
endif ()

## SETUP
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

if ( LUA_WIN )
  # Add extra rc files to the windows build
  if ( MSVC OR MINGW )
    set ( LUA_DEF src/lua.def )
    set ( LUA_DLL_RC src/lua_dll.rc )
    set ( LUA_RC src/lua.rc )
    set ( LUAC_RC src/luac.rc )
  endif ()
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

file ( GLOB SRC_LIB src/lib_*.c )
if ( LUA_USE_RELATIVE_LOADLIB )
  list ( REMOVE_ITEM SRC_LIB ${CMAKE_CURRENT_SOURCE_DIR}/src/lib_package.c )
else ()
  list ( REMOVE_ITEM SRC_LIB ${CMAKE_CURRENT_SOURCE_DIR}/src/lib_package_rel.c )
endif ()

file ( GLOB SRC_BUILD src/buildvm*.c )
file ( GLOB SRC_JIT src/lj_*.c )

# Optionally rebuild buildvm_*.h header.
if (LUAJIT_RUN_DYNASM)
  set(DASM ${LUA} ${CMAKE_CURRENT_SOURCE_DIR}/dynasm/dynasm.lua)
  set(DASM_FLAGS_X86)
  set(DASM_FLAGS_X64 -D X64)
  set(DASM_FLAGS_X64WIN -D X64 -D X64WIN)
  set(DASM_FLAGS_ARM)
  set(DASM_FLAGS_PPCSPE -D SPE)
  add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/buildvm_x86.h
    COMMAND ${DASM} -LN ${DASM_FLAGS_X86} -o ${CMAKE_CURRENT_BINARY_DIR}/buildvm_x86.h ${CMAKE_CURRENT_SOURCE_DIR}/src/buildvm_x86.dasc
    DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/dynasm/dynasm.lua
  )
  add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/buildvm_x64.h
    COMMAND ${DASM} -LN ${DASM_FLAGS_X64} -o ${CMAKE_CURRENT_BINARY_DIR}/buildvm_x64.h ${CMAKE_CURRENT_SOURCE_DIR}/src/buildvm_x86.dasc
    DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/dynasm/dynasm.lua
  )
  add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/buildvm_x64win.h
    COMMAND ${DASM} -LN ${DASM_FLAGS_X64WIN} -o ${CMAKE_CURRENT_BINARY_DIR}/buildvm_x64win.h ${CMAKE_CURRENT_SOURCE_DIR}/src/buildvm_x86.dasc
    DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/dynasm/dynasm.lua
  )
  add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/buildvm_arm.h
    COMMAND ${DASM} -LN ${DASM_FLAGS_PPCSPE} -o ${CMAKE_CURRENT_BINARY_DIR}/buildvm_arm.h ${CMAKE_CURRENT_SOURCE_DIR}/src/buildvm_arm.dasc
    DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/dynasm/dynasm.lua
  )
  add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/buildvm_ppcspe.h
    COMMAND ${DASM} -LN ${DASM_FLAGS_PPCSPE} -o ${CMAKE_CURRENT_BINARY_DIR}/buildvm_ppcspe.h ${CMAKE_CURRENT_SOURCE_DIR}/src/buildvm_ppc.dasc
    DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/dynasm/dynasm.lua
  )
  
  # Note: not all of these are needed for the current platform, but for simplicity
  # specify all.
  set_source_files_properties(src/buildvm.c PROPERTIES OBJECT_DEPENDS
	${CMAKE_CURRENT_BINARY_DIR}/buildvm_x86.h
	${CMAKE_CURRENT_BINARY_DIR}/buildvm_x64.h
	${CMAKE_CURRENT_BINARY_DIR}/buildvm_x64win.h
	${CMAKE_CURRENT_BINARY_DIR}/buildvm_arm.h
	${CMAKE_CURRENT_BINARY_DIR}/buildvm_ppcspe.h)
  include_directories(${CMAKE_CURRENT_BINARY_DIR})
endif ()

## GENERATE
add_executable ( buildvm ${SRC_BUILD} )

macro(add_buildvm_target _target _mode)
  add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${_target}
    COMMAND buildvm ARGS -m ${_mode} -o ${CMAKE_CURRENT_BINARY_DIR}/${_target} ${ARGN}
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    DEPENDS buildvm ${ARGN}
  )
endmacro(add_buildvm_target)

if (MSVC)
  add_buildvm_target ( lj_vm.obj peobj )
  set (LJ_VM_SRC ${CMAKE_CURRENT_BINARY_DIR}/lj_vm.obj)
else ()
  add_buildvm_target ( lj_vm.s ${LJVM_MODE} )
  set (LJ_VM_SRC ${CMAKE_CURRENT_BINARY_DIR}/lj_vm.s)
endif ()
add_buildvm_target ( lj_ffdef.h   ffdef   ${SRC_LIB} )
add_buildvm_target ( lj_bcdef.h  bcdef  ${SRC_LIB} )
add_buildvm_target ( lj_folddef.h folddef src/lj_opt_fold.c )
add_buildvm_target ( lj_recdef.h  recdef  ${SRC_LIB} )
add_buildvm_target ( lj_libdef.h  libdef  ${SRC_LIB} )

SET(DEPS 
  ${LJ_VM_SRC}
  ${CMAKE_CURRENT_BINARY_DIR}/lj_ffdef.h
  ${CMAKE_CURRENT_BINARY_DIR}/lj_bcdef.h
  ${CMAKE_CURRENT_BINARY_DIR}/lj_libdef.h
  ${CMAKE_CURRENT_BINARY_DIR}/lj_recdef.h
  ${CMAKE_CURRENT_BINARY_DIR}/lj_folddef.h 
  )
                  
## COMPILE
include_directories ( ${CMAKE_CURRENT_BINARY_DIR} dynasm src )
add_library ( liblua SHARED ${SRC_JIT} ${SRC_LIB} src/lua_dll.rc ${DEPS} ) #old: src/lua.def
target_link_libraries ( liblua ${LIBS} )
set_target_properties ( liblua PROPERTIES OUTPUT_NAME lua CLEAN_DIRECT_OUTPUT 1 )

add_executable ( lua src/luajit.c src/lua.rc )
target_link_libraries ( lua liblua )

# On windows a variant of the lua interpreter without console output needs to be built
if(LUA_BUILD_WLUA)
  add_executable ( wlua WIN32 src/luajit.c src/lua.rc )
  target_link_libraries ( wlua liblua )  
  install_executable ( wlua )
endif()

## INSTALL
install_executable( lua )
install_library( liblua )
install_header( src/lua.h ${CMAKE_CURRENT_BINARY_DIR}/luaconf.h src/lualib.h src/lauxlib.h )

install_lua_module ( strict etc/strict.lua )
install_doc ( doc/ )
install_foo ( etc/ )
install_data ( COPYRIGHT README.md )
install_data ( lib/ INTO lib )

