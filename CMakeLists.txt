
 # ***** BEGIN LICENSE BLOCK *****
 # Version: MPL 1.1/GPL 2.0/LGPL 2.1
 #
 # The contents of this file are subject to the Mozilla Public License Version
 # 1.1 (the "License"); you may not use this file except in compliance with
 # the License. You may obtain a copy of the License at
 # http://www.mozilla.org/MPL/
 #
 # Software distributed under the License is distributed on an "AS IS" basis,
 # WITHOUT WARRANTY OF ANY KIND, either express or implied. See the License
 # for the specific language governing rights and limitations under the
 # License.
 #
 # The Original Code is cmgui
 #
 # The Initial Developer of the Original Code is
 # Auckland Uniservices Ltd, Auckland, New Zealand.
 # Portions created by the Initial Developer are Copyright (C) 2005
 # the Initial Developer. All Rights Reserved.
 #
 # Contributor(s): 
 #
 # Alternatively, the contents of this file may be used under the terms of
 # either the GNU General Public License Version 2 or later (the "GPL"), or
 # the GNU Lesser General Public License Version 2.1 or later (the "LGPL"),
 # in which case the provisions of the GPL or the LGPL are applicable instead
 # of those above. If you wish to allow use of your version of this file only
 # under the terms of either the GPL or the LGPL, and not to allow others to
 # use your version of this file under the terms of the MPL, indicate your
 # decision by deleting the provisions above and replace them with the notice
 # and other provisions required by the GPL or the LGPL. If you do not delete
 # the provisions above, a recipient may use your version of this file under
 # the terms of any one of the MPL, the GPL or the LGPL.
 #
 # ***** END LICENSE BLOCK ***** */

# Set the minimum version of cmake required to 2.6
cmake_minimum_required(VERSION 3.0)
project(Glew VERSION 1.5.5 LANGUAGES C CXX)

SET( LIBRARY_TARGET_NAME glew )
STRING( TOUPPER ${LIBRARY_TARGET_NAME} UPPERCASE_LIBRARY_TARGET_NAME )

SET( GLEW_MAJOR_VERSION "1" )
SET( GLEW_MINOR_VERSION "5" )
SET( GLEW_PATCH_VERSION "5" )

# Set up library options
# Generic library options
OPTION( ${UPPERCASE_LIBRARY_TARGET_NAME}_BUILD_STATIC_LIB "Build static glew" TRUE )
OPTION( ${UPPERCASE_LIBRARY_TARGET_NAME}_INSTALL_CONFIG "Install a config file for the library" FALSE )

IF( ${UPPERCASE_LIBRARY_TARGET_NAME}_BUILD_STATIC_LIB )
	ADD_DEFINITIONS( -DGLEW_STATIC )
ELSE( ${UPPERCASE_LIBRARY_TARGET_NAME}_BUILD_STATIC_LIB )
	ADD_DEFINITIONS( -DGLEW_BUILD )
ENDIF( ${UPPERCASE_LIBRARY_TARGET_NAME}_BUILD_STATIC_LIB )
IF( MSVC )
	ADD_DEFINITIONS( -D_CRT_SECURE_NO_WARNINGS )
ENDIF( MSVC )
IF( ${UPPERCASE_LIBRARY_TARGET_NAME}_BUILD_STATIC_LIB )
	SET( LIBRARY_BUILD_TYPE STATIC )
	SET( LIBRARY_INSTALL_TYPE ARCHIVE )
ELSE( ${UPPERCASE_LIBRARY_TARGET_NAME}_BUILD_STATIC_LIB )
	SET( LIBRARY_BUILD_TYPE SHARED )
	SET( LIBRARY_INSTALL_TYPE LIBRARY )
	IF( WIN32 )
		SET( LIBRARY_INSTALL_TYPE RUNTIME )
	ENDIF( WIN32 )
ENDIF( ${UPPERCASE_LIBRARY_TARGET_NAME}_BUILD_STATIC_LIB )

SET( ${UPPERCASE_LIBRARY_TARGET_NAME}_SRCS
	src/glew.c )
SET( ${UPPERCASE_LIBRARY_TARGET_NAME}_PUBLIC_HDRS
	include/GL/glew.h
	include/GL/glxew.h
	include/GL/wglew.h )

SET( CMAKE_DEBUG_POSTFIX "d" )
SET( CMAKE_MFC_FLAG 0 )
INCLUDE_DIRECTORIES( include )
IF( WIN32 AND NOT ${UPPERCASE_LIBRARY_TARGET_NAME}_BUILD_STATIC_LIB )
	SET( ${UPPERCASE_LIBRARY_TARGET_NAME}_WIN32_XTRAS 
		win32/glew.def
		win32/glew1.rc )
ENDIF()

add_library( ${LIBRARY_TARGET_NAME} ${LIBRARY_BUILD_TYPE} 
	${${UPPERCASE_LIBRARY_TARGET_NAME}_SRCS}
	${${UPPERCASE_LIBRARY_TARGET_NAME}_PUBLIC_HDRS}
	${${UPPERCASE_LIBRARY_TARGET_NAME}_WIN32_XTRAS} )

# Install targets
IF( WIN32 AND NOT ${UPPERCASE_LIBRARY_TARGET_NAME}_BUILD_STATIC_LIB )
	SET_TARGET_PROPERTIES(${LIBRARY_TARGET_NAME} PROPERTIES IMPORT_SUFFIX _dll.lib)
	INSTALL( TARGETS ${LIBRARY_TARGET_NAME} ARCHIVE
		DESTINATION lib )
endif()
if (ENABLE_PIC)
    set_target_properties(${LIBRARY_TARGET_NAME} PROPERTIES POSITION_INDEPENDENT_CODE ON)
endif()

install(TARGETS ${LIBRARY_TARGET_NAME}
    EXPORT glew-config ${LIBRARY_INSTALL_TYPE}
	DESTINATION lib
	INCLUDES DESTINATION include/GL)
install(FILES ${${UPPERCASE_LIBRARY_TARGET_NAME}_PUBLIC_HDRS}
	DESTINATION include/GL)
install(EXPORT glew-config 
	DESTINATION ${PACKAGE_CONFIG_DIR})	
include(CMakePackageConfigHelpers)
WRITE_BASIC_PACKAGE_VERSION_FILE(${CMAKE_CURRENT_BINARY_DIR}/glew-config-version.cmake
    COMPATIBILITY AnyNewerVersion)
install(FILES 
    ${CMAKE_CURRENT_BINARY_DIR}/glew-config-version.cmake
    #${CMAKE_CURRENT_BINARY_DIR}/glew-config-dependencies.cmake
    DESTINATION ${PACKAGE_CONFIG_DIR})

