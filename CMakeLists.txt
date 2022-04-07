if ( NOT DEFINED CMAKE_BUILD_TYPE )
  set( CMAKE_BUILD_TYPE Release CACHE STRING "Build type" )
endif ()

if(POLICY CMP0003)
  cmake_policy (SET CMP0003 NEW)
endif ()

if(POLICY CMP0042)
  cmake_policy (SET CMP0042 NEW)
endif ()

if(POLICY CMP0048)
  cmake_policy(SET CMP0048 NEW)
endif ()

project (glew C)

cmake_minimum_required (VERSION 3.10)

include(GNUInstallDirs)

set(PACKAGE_CONFIG_DIR "share/cmake/GLEW" CACHE STRING "Directory for package config files (relative to CMAKE_INSTALL_PREFIX).")

set(CMAKE_DEBUG_POSTFIX d)
list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/build/cmake)

option (BUILD_UTILS "utilities" ON)
option (GLEW_REGAL "Regal mode" OFF)
option (GLEW_BUILD_OSMESA "OSMesa mode" OFF)
option (GLEW_BUILD_STATIC "Static library" ON)
option (GLEW_BUILD_SHARED "Shared library" OFF)
if (APPLE)
  option (BUILD_FRAMEWORK "Build Framework bundle for OSX" OFF)
endif ()

if (NOT GLEW_BUILD_STATIC AND NOT GLEW_BUILD_SHARED)
  message(FATAL_ERROR "Must build either share or static library.")
endif ()

set(GLEW_DIR ${CMAKE_CURRENT_SOURCE_DIR})

# get version from config/version
file (STRINGS ${GLEW_DIR}/config/version  _VERSION_MAJOR_STRING REGEX "GLEW_MAJOR[ ]*=[ ]*[0-9]+.*")
string (REGEX REPLACE "GLEW_MAJOR[ ]*=[ ]*([0-9]+)" "\\1" CPACK_PACKAGE_VERSION_MAJOR ${_VERSION_MAJOR_STRING})
file (STRINGS ${GLEW_DIR}/config/version  _VERSION_MINOR_STRING REGEX "GLEW_MINOR[ ]*=[ ]*[0-9]+.*")
string (REGEX REPLACE "GLEW_MINOR[ ]*=[ ]*([0-9]+)" "\\1" CPACK_PACKAGE_VERSION_MINOR ${_VERSION_MINOR_STRING})
file (STRINGS ${GLEW_DIR}/config/version  _VERSION_PATCH_STRING REGEX "GLEW_MICRO[ ]*=[ ]*[0-9]+.*")
string (REGEX REPLACE "GLEW_MICRO[ ]*=[ ]*([0-9]+)" "\\1" CPACK_PACKAGE_VERSION_PATCH ${_VERSION_PATCH_STRING})
set (GLEW_VERSION ${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH})

set (CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set (CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set (CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

if (GLEW_BUILD_OSMESA)
  find_package (OSMesa REQUIRED)
else ()
  find_package (OpenGL REQUIRED)
endif ()

# X11 required except for Windows and Apple OSX platforms
if (NOT WIN32 AND NOT APPLE)
  find_package (X11)
endif()

if (WIN32)
  set (GLEW_LIB_NAME glew32)
else ()
  set (GLEW_LIB_NAME GLEW)
  set (DLL_PREFIX lib)
endif ()

set (GLEW_LIBRARIES ${OPENGL_LIBRARIES} ${X11_LIBRARIES})

add_definitions (-DGLEW_NO_GLU)

#### Regal mode ####

if (GLEW_REGAL)
  if (WIN32)
    set (REGAL_LIB_NAME regal32)
  else ()
    set (REGAL_LIB_NAME Regal)
  endif ()
  add_definitions (-DGLEW_REGAL)
  set (GLEW_LIBRARIES ${REGAL_LIB_NAME})
endif ()

#### OSMesa mode ####

if (GLEW_BUILD_OSMESA)
  set (GLEW_OSMESA_LIBRARIES ${OSMESA_LIB_NAME})
endif ()

#### EGL ####

if (GLEW_EGL AND UNIX)
  add_definitions (-DGLEW_EGL)
  if (OpenGL::EGL)
    message (FATAL_ERROR "EGL library set but not found.")
  endif()
  set (GLEW_LIBRARIES ${OPENGL_LIBRARIES} ${OPENGL_egl_LIBRARY})
endif ()

#### GLEW ####

include_directories (${GLEW_DIR}/include ${X11_INCLUDE_DIR})

set (GLEW_PUBLIC_HEADERS_FILES ${GLEW_DIR}/include/GL/wglew.h ${GLEW_DIR}/include/GL/glew.h ${GLEW_DIR}/include/GL/glxew.h)
set (GLEW_SRC_FILES ${GLEW_DIR}/src/glew.c)

if (WIN32)
  list (APPEND GLEW_SRC_FILES ${GLEW_DIR}/build/glew.rc)
endif ()

if (GLEW_BUILD_SHARED)
  add_library (glew SHARED ${GLEW_PUBLIC_HEADERS_FILES} ${GLEW_SRC_FILES})
  set_target_properties (glew PROPERTIES COMPILE_DEFINITIONS "GLEW_BUILD" OUTPUT_NAME "${GLEW_LIB_NAME}" PREFIX "${DLL_PREFIX}"
                                         VERSION ${GLEW_VERSION}
                                         SOVERSION ${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR})
endif ()

if (GLEW_BUILD_STATIC)
  add_library (glew_s STATIC ${GLEW_PUBLIC_HEADERS_FILES} ${GLEW_SRC_FILES})
  set_target_properties (glew_s PROPERTIES COMPILE_DEFINITIONS "GLEW_STATIC" OUTPUT_NAME "${GLEW_LIB_NAME}" PREFIX lib)
endif ()

if (GLEW_BUILD_OSMESA)
  add_library (glew_s_osmesa STATIC ${GLEW_PUBLIC_HEADERS_FILES} ${GLEW_SRC_FILES})
  set_target_properties (glew_s_osmesa PROPERTIES COMPILE_DEFINITIONS "GLEW_STATIC;GLEW_OSMESA" OUTPUT_NAME "${GLEW_LIB_NAME}-osmesa" PREFIX lib)
endif ()

if (MSVC)
  # add options from visual studio project
  if (GLEW_BUILD_SHARED)
    target_compile_definitions (glew PRIVATE "GLEW_BUILD;VC_EXTRALEAN")
    target_link_libraries (glew LINK_PRIVATE -BASE:0x62AA0000)
    target_compile_options (glew PRIVATE -GS-)
    target_link_libraries (glew LINK_PRIVATE -nodefaultlib -noentry)
    target_link_libraries (glew LINK_PRIVATE libvcruntime.lib)
    target_link_libraries (glew LINK_PRIVATE msvcrt.lib )
  endif ()
  if (GLEW_BUILD_STATIC)
    target_compile_definitions (glew_s PRIVATE "GLEW_STATIC;VC_EXTRALEAN")
    # kill security checks which are dependent on stdlib
    target_compile_options (glew_s PRIVATE -GS-)
  endif ()
  if (GLEW_BUILD_OSMESA)
    target_compile_definitions (glew_s_osmesa PRIVATE "VC_EXTRALEAN")
    # kill security checks which are dependent on stdlib
    target_compile_options (glew_s_osmesa PRIVATE -GS-)
  endif ()
  # remove stdlib dependency
  string(REGEX REPLACE "/RTC(su|[1su])" "" CMAKE_C_FLAGS_DEBUG ${CMAKE_C_FLAGS_DEBUG})
elseif (WIN32 AND ((CMAKE_C_COMPILER_ID MATCHES "GNU") OR (CMAKE_C_COMPILER_ID MATCHES "Clang")))
  # remove stdlib dependency on windows with GCC and Clang (for similar reasons
  # as to MSVC - to allow it to be used with any Windows compiler)
  if (GLEW_BUILD_SHARED)
    target_compile_options (glew PRIVATE -fno-builtin -fno-stack-protector)
    target_link_libraries (glew LINK_PRIVATE -nostdlib)
  endif ()
  if (GLEW_BUILD_STATIC)
    target_compile_options (glew_s PRIVATE -fno-builtin -fno-stack-protector)
  endif ()
  if (GLEW_BUILD_OSMESA)
    target_compile_options (glew_s_osmesa PRIVATE -fno-builtin -fno-stack-protector)
  endif ()
endif ()

if (BUILD_FRAMEWORK AND GLEW_BUILD_SHARED)
  set_target_properties(glew PROPERTIES
    FRAMEWORK TRUE
    FRAMEWORK_VERSION ${GLEW_VERSION}
    MACOSX_FRAMEWORK_IDENTIFIER net.sourceforge.glew
    MACOSX_FRAMEWORK_SHORT_VERSION_STRING ${GLEW_VERSION}
    MACOSX_FRAMEWORK_BUNDLE_VERSION ${GLEW_VERSION}
    XCODE_ATTRIBUTE_INSTALL_PATH "@rpath"
    PUBLIC_HEADER "${GLEW_PUBLIC_HEADERS_FILES}"
    OUTPUT_NAME GLEW
  )
endif()

if (GLEW_BUILD_SHARED)
  target_link_libraries (glew LINK_PUBLIC ${GLEW_LIBRARIES})
  target_include_directories(glew PUBLIC $<INSTALL_INTERFACE:include>)
  set_target_properties(glew PROPERTIES POSITION_INDEPENDENT_CODE ON)
endif ()
if (GLEW_BUILD_STATIC)
  target_link_libraries (glew_s ${GLEW_LIBRARIES})
  target_compile_definitions(glew_s INTERFACE "GLEW_STATIC")
  target_include_directories(glew_s PUBLIC $<INSTALL_INTERFACE:include>)
  set_target_properties(glew_s PROPERTIES POSITION_INDEPENDENT_CODE ON)
endif ()
if (GLEW_BUILD_OSMESA)
  target_link_libraries (glew_s_osmesa ${OSMesa_LIBRARIES})
  target_compile_definitions(glew_s_osmesa INTERFACE "GLEW_STATIC")
  target_include_directories(glew_s_osmesa PUBLIC $<INSTALL_INTERFACE:include>)
  set_target_properties(glew_s_osmesa PROPERTIES POSITION_INDEPENDENT_CODE ON)
endif ()

set(targets_to_install "")
if(GLEW_BUILD_SHARED)
  list(APPEND targets_to_install glew)
endif()

if(GLEW_BUILD_STATIC)
  list(APPEND targets_to_install glew_s)
endif()

if(GLEW_BUILD_OSMESA)
  list(APPEND targets_to_install glew_s_osmesa)
endif()

install ( TARGETS ${targets_to_install}
          EXPORT glew-targets
          RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
          LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
          ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
          FRAMEWORK DESTINATION ${CMAKE_INSTALL_PREFIX}
)

if (BUILD_UTILS)
  set (GLEWINFO_SRC_FILES ${GLEW_DIR}/src/glewinfo.c)
  if (WIN32)
    list (APPEND GLEWINFO_SRC_FILES ${GLEW_DIR}/build/glewinfo.rc)
  endif ()
  add_executable (glewinfo ${GLEWINFO_SRC_FILES})
  if(GLEW_BUILD_SHARED)
     target_link_libraries (glewinfo glew)
  else()
     target_link_libraries (glewinfo glew_s)
  endif()
  if (NOT WIN32)
    target_link_libraries(glewinfo ${X11_LIBRARIES})
  endif ()

  set (VISUALINFO_SRC_FILES ${GLEW_DIR}/src/visualinfo.c)
  if (WIN32)
    list (APPEND VISUALINFO_SRC_FILES ${GLEW_DIR}/build/visualinfo.rc)
  endif ()
  add_executable (visualinfo ${VISUALINFO_SRC_FILES})
  if(GLEW_BUILD_SHARED)
     target_link_libraries (visualinfo glew)
  else()
     target_link_libraries (visualinfo glew_s)
  endif()
  if (NOT WIN32)
    target_link_libraries(visualinfo ${X11_LIBRARIES})
  endif ()

  install (TARGETS glewinfo visualinfo
           DESTINATION ${CMAKE_INSTALL_BINDIR})
endif ()

set (prefix ${CMAKE_INSTALL_PREFIX})
set (exec_prefix ${CMAKE_INSTALL_PREFIX})
set (libdir ${CMAKE_INSTALL_FULL_LIBDIR})
set (includedir ${CMAKE_INSTALL_FULL_INCLUDEDIR})
set (version ${GLEW_VERSION})
set (libname ${GLEW_LIB_NAME})
set (cflags)
set (requireslib glu)

#  Mac OSX has no glu.pc unless optional X11/GLX is installed
if (APPLE)
  set (requireslib)
endif ()

configure_file (${GLEW_DIR}/glew.pc.in ${CMAKE_CURRENT_BINARY_DIR}/glew.pc @ONLY)

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/glew.pc
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
)

if(WIN32 AND MSVC AND (NOT MSVC_VERSION LESS 1600) AND (NOT CMAKE_VERSION VERSION_LESS "3.1"))
  install(
    FILES $<TARGET_PDB_FILE:glew>
    DESTINATION ${CMAKE_INSTALL_LIBDIR}
    CONFIGURATIONS Debug RelWithDebInfo
  )
endif()

install (FILES
  ${GLEW_DIR}/include/GL/wglew.h
  ${GLEW_DIR}/include/GL/glew.h
  ${GLEW_DIR}/include/GL/glxew.h
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/GL)

install(EXPORT glew-targets DESTINATION ${PACKAGE_CONFIG_DIR}
  NAMESPACE GLEW::)
install(FILES
  ${CMAKE_CURRENT_SOURCE_DIR}/build/cmake/glew-config.cmake
  ${CMAKE_CURRENT_SOURCE_DIR}/build/cmake/CopyImportedTargetProperties.cmake
  DESTINATION ${PACKAGE_CONFIG_DIR})

if(NOT TARGET uninstall)
  configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/build/cmake/cmake_uninstall.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/build/cmake/cmake_uninstall.cmake
    IMMEDIATE @ONLY)

  add_custom_target(uninstall
    COMMAND ${CMAKE_COMMAND} -P
    ${CMAKE_CURRENT_BINARY_DIR}/build/cmake/cmake_uninstall.cmake)
endif()
