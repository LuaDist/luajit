# Copyright (C) 2007-2009 LuaDist.
# Created by Peter Drahoš
# Redistribution and use of this file is allowed according to the terms of the MIT license.
# For details see the COPYRIGHT file distributed with LuaDist.
# Please note that the package source code is licensed under its own license.

PROJECT ( luajit C)
CMAKE_MINIMUM_REQUIRED(VERSION 2.8)
INCLUDE(dist.cmake )

# Determine install host
IF ( WIN32 AND NOT CYGWIN)
	ADD_DEFINITIONS ( -DLUA_BUILD_AS_DLL )
ELSE ( )
	ADD_DEFINITIONS ( -DLUA_USE_POSIX -DLUA_USE_DLOPEN )
	SET ( LIBS m dl )
ENDIF ( )

# Add Readline support when available
FIND_PATH(READLINE_INCLUDE_DIR readline/readline.h)
FIND_LIBRARY(READLINE_LIBRARY NAMES readline)
IF ( READLINE_LIBRARY )
	INCLUDE_DIRECTORIES ( ${READLINE_INCLUDE_DIR} )
	ADD_DEFINITIONS ( -DLUA_USE_READLINE )
	SET ( LIBS ${LIBS} ${READLINE_LIBRARY} )
ENDIF ( )

# Add Curses support when available
INCLUDE(FindCurses)
IF ( CURSES_LIBRARY )
	INCLUDE_DIRECTORIES ( ${CURSES_INCLUDE_DIR} )
	SET ( LIBS ${LIBS} ${CURSES_LIBRARY} )
ENDIF ( )

# Build Libraries
SET(SRC_COCO src/lcoco.c)
SET(SRC_LIBJIT src/ljit_core.c src/ljit_mem.c src/ljit_dasm.c src/ljit_backend.c)
SET(SRC_JITCORE src/lapi.c src/lcode.c src/ldebug.c src/ldo.c src/ldump.c src/lfunc.c src/lgc.c src/llex.c src/lmem.c src/lobject.c src/lopcodes.c src/lparser.c src/lstate.c src/lstring.c src/ltable.c src/ltm.c src/lundump.c src/lvm.c src/lzio.c ${SRC_COCO} ${SRC_LIBJIT})
SET(SRC_LIBLUAJIT src/lauxlib.c src/lbaselib.c src/ldblib.c src/liolib.c src/lmathlib.c src/loslib.c src/ltablib.c src/lstrlib.c src/loadlib.c src/linit.c src/ljitlib.c)

INCLUDE_DIRECTORIES (dynasm src)
ADD_LIBRARY ( liblua SHARED ${SRC_JITCORE} ${SRC_LIBLUAJIT} src/lua.def src/lua_dll.rc)
TARGET_LINK_LIBRARIES ( liblua ${LIBS})
SET_TARGET_PROPERTIES ( liblua PROPERTIES OUTPUT_NAME lua CLEAN_DIRECT_OUTPUT 1 )

ADD_LIBRARY ( liblua_static ${SRC_JITCORE} ${SRC_LIBLUAJIT})
TARGET_LINK_LIBRARIES ( liblua_static ${LIBS})

# Build Executables
SET ( SRC_LUA src/lua.c )
SET ( SRC_LUAC src/luac.c src/print.c )

ADD_EXECUTABLE ( lua ${SRC_LUA} src/lua.rc)
ADD_EXECUTABLE ( luac ${SRC_LUAC} src/lua_simple.rc)
TARGET_LINK_LIBRARIES ( lua liblua )
TARGET_LINK_LIBRARIES ( luac liblua_static )

# Install
INSTALL ( TARGETS lua luac RUNTIME DESTINATION ${INSTALL_BIN} )
INSTALL ( TARGETS liblua DESTINATION ${INSTALL_LIB} )
INSTALL ( FILES src/lua.h src/luaconf.h src/lualib.h src/lauxlib.h etc/lua.hpp DESTINATION ${INSTALL_INC} )
INSTALL ( FILES etc/strict.lua DESTINATION ${INSTALL_LMOD} )
INSTALL ( DIRECTORY jit DESTINATION ${INSTALL_LMOD} )
INSTALL ( DIRECTORY doc etc DESTINATION ${INSTALL_DATA} PATTERN ".svn" EXCLUDE )
INSTALL ( DIRECTORY jitdoc DESTINATION ${INSTALL_DOC} PATTERN ".svn" EXCLUDE )
INSTALL ( FILES README COPYRIGHT DESTINATION ${INSTALL_DATA} )

