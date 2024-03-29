################################################################################
# Project:  Lib webp
# Purpose:  CMake build script from original CMakeLists.txt.
# Author:   Dmitry Baryshnikov, dmitry.baryshnikov@nexgis.com
################################################################################
# Copyright (C) 2017-2018, NextGIS <info@nextgis.com>
# Copyright (c) 2010, Google Inc
################################################################################
cmake_minimum_required(VERSION 2.8.12 FATAL_ERROR)

project(webp C)

if(NOT DEFINED PACKAGE_VENDOR)
    set(PACKAGE_VENDOR NextGIS)
endif()

if(NOT DEFINED PACKAGE_BUGREPORT)
    set(PACKAGE_BUGREPORT info@nextgis.com)
endif()

# Some init settings
set(CMAKE_COLOR_MAKEFILE ON)
# Set path to additional CMake modules
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

if(OSX_FRAMEWORK AND (BUILD_SHARED_LIBS OR BUILD_STATIC_LIBS))
  message(FATAL_ERROR "Only OSX_FRAMEWORK key or any or both BUILD_SHARED_LIBS
                       and BUILD_STATIC_LIBS keys are permitted")
endif()

if(OSX_FRAMEWORK)
    set(INSTALL_BIN_DIR "bin" CACHE INTERNAL "Installation directory for executables" FORCE)
    set(INSTALL_LIB_DIR "Library/Frameworks" CACHE INTERNAL "Installation directory for libraries" FORCE)
    set(SKIP_INSTALL_EXECUTABLES OFF)
    set(SKIP_INSTALL_EXPORT ON)
    set(SKIP_INSTALL_FILES OFF)
    set(SKIP_INSTALL_DOCS OFF)
    set(CMAKE_MACOSX_RPATH ON)
    set(SKIP_INSTALL_HEADERS ON)
    set(LIB_TYPE SHARED)
else()
    include(GNUInstallDirs)
    set(INSTALL_BIN_DIR ${CMAKE_INSTALL_BINDIR} CACHE INTERNAL "Installation directory for executables" FORCE)
    set(INSTALL_LIB_DIR ${CMAKE_INSTALL_LIBDIR} CACHE INTERNAL "Installation directory for libraries" FORCE)
    set(INSTALL_DOC_DIR ${CMAKE_INSTALL_DOCDIR} CACHE INTERNAL "Installation directory for doc pages" FORCE)
    if(BUILD_SHARED_LIBS)
        set(LIB_TYPE SHARED)
    else()
        set(LIB_TYPE STATIC)
    endif()
endif()

include(util)
check_version(WEBP_VERSION_MAJOR WEBP_VERSION_MINOR WEBP_VERSION_BUILD)
set(VERSION ${WEBP_VERSION_MAJOR}.${WEBP_VERSION_MINOR}.${WEBP_VERSION_BUILD})
set(PROJECT_VERSION ${VERSION})

if(OSX_FRAMEWORK)
    set(FRAMEWORK_VERSION "${WEBP_VERSION_MAJOR}")
    set(INSTALL_DOC_DIR "${INSTALL_LIB_DIR}/${PROJECT_NAME}.framework/Versions/${FRAMEWORK_VERSION}/Resources/docs")
endif()

report_version(${PROJECT_NAME} ${VERSION})

# Options for coder / decoder executables.
option(WEBP_ENABLE_SIMD "Enable any SIMD optimization." ON)
option(WEBP_BUILD_ANIM_UTILS "Build animation utilities." ON)
option(WEBP_BUILD_CWEBP "Build the cwebp command line tool." ON)
option(WEBP_BUILD_DWEBP "Build the dwebp command line tool." ON)
option(WEBP_BUILD_GIF2WEBP "Build the gif2webp conversion tool." ON)
option(WEBP_BUILD_IMG2WEBP "Build the img2webp animation tool." ON)
option(WEBP_BUILD_VWEBP "Build the vwebp viewer tool." ON)
option(WEBP_BUILD_WEBPINFO "Build the webpinfo command line tool." ON)
option(WEBP_BUILD_WEBPMUX "Build the webpmux command line tool." ON)
option(WEBP_BUILD_EXTRAS "Build extras." ON)
option(WEBP_BUILD_WEBP_JS "Emscripten build of webp.js." OFF)
option(WEBP_NEAR_LOSSLESS "Enable near-lossless encoding" ON)
option(WEBP_ENABLE_SWAP_16BIT_CSP "Enable byte swap for 16 bit colorspaces." OFF)
option(WEBP_BUILD_LIBWEBPMUX "Build the libwebpmux library." ON)
option(WEBP_USE_THREAD "Enable threading support" ON)
set(WEBP_BITTRACE "0" CACHE STRING "Bit trace mode (0=none, 1=bit, 2=bytes)")
set_property(CACHE WEBP_BITTRACE PROPERTY STRINGS 0 1 2)

# Option needed for handling Unicode file names on Windows.
if(WIN32)
  option(WEBP_UNICODE "Build Unicode executables." ON)
endif()

if(WEBP_BUILD_WEBP_JS)
  set(WEBP_BUILD_ANIM_UTILS OFF)
  set(WEBP_BUILD_CWEBP OFF)
  set(WEBP_BUILD_DWEBP OFF)
  set(WEBP_BUILD_GIF2WEBP OFF)
  set(WEBP_BUILD_IMG2WEBP OFF)
  set(WEBP_BUILD_VWEBP OFF)
  set(WEBP_BUILD_WEBPINFO OFF)
  set(WEBP_BUILD_WEBPMUX OFF)
  set(WEBP_BUILD_EXTRAS OFF)
  set(WEBP_USE_THREAD OFF)

  if(WEBP_ENABLE_SIMD)
    message("wasm2js does not support SIMD, disabling webp.js generation.")
  endif()
endif()

set(WEBP_DEP_LIBRARIES)
set(WEBP_DEP_INCLUDE_DIRS)

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE "Release"
      CACHE STRING "Build type: Release, Debug, MinSizeRel or RelWithDebInfo"
      FORCE)
endif()

# Include dependencies.
include(cmake/deps_borsch.cmake)
# include(GNUInstallDirs)

# ##############################################################################
# Options.
if(WEBP_ENABLE_SWAP_16BIT_CSP)
  add_definitions(-DWEBP_SWAP_16BIT_CSP=1)
endif()

if(NOT WEBP_BITTRACE STREQUAL "0")
  add_definitions(-DBITTRACE=${WEBP_BITTRACE})
endif()

if(WEBP_UNICODE)
  # Windows recommends setting both UNICODE and _UNICODE.
  add_definitions(-DUNICODE -D_UNICODE)
endif()

# set(prefix ${CMAKE_INSTALL_PREFIX})
# set(exec_prefix "\$\{prefix\}")
# set(libdir "\$\{prefix\}/lib")
# set(includedir "\$\{prefix\}/include")
set(PTHREAD_LIBS ${CMAKE_THREAD_LIBS_INIT})
set(INSTALLED_LIBRARIES)

# ##############################################################################
# Android only.
if(ANDROID)
  include_directories(${ANDROID_NDK}/sources/android/cpufeatures)
  add_library(cpufeatures STATIC
              ${ANDROID_NDK}/sources/android/cpufeatures/cpu-features.c)
  list(APPEND INSTALLED_LIBRARIES cpufeatures)
  target_link_libraries(cpufeatures dl)
  set(WEBP_DEP_LIBRARIES ${WEBP_DEP_LIBRARIES} cpufeatures)
  set(WEBP_DEP_INCLUDE_DIRS ${WEBP_DEP_INCLUDE_DIRS}
      ${ANDROID_NDK}/sources/android/cpufeatures)
  add_definitions(-DHAVE_CPU_FEATURES_H=1)
  set(HAVE_CPU_FEATURES_H 1)
else()
  set(HAVE_CPU_FEATURES_H 0)
endif()

# function(configure_pkg_config FILE)
#   configure_file("${CMAKE_CURRENT_SOURCE_DIR}/${FILE}.in"
#                  "${CMAKE_CURRENT_BINARY_DIR}/${FILE}")
#
#   if(HAVE_MATH_LIBRARY)
#     # MSVC doesn't have libm
#     file(READ ${CMAKE_CURRENT_BINARY_DIR}/${FILE} data)
#     string(REPLACE "-lm" "" data ${data})
#     file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/${FILE} ${data})
#   endif()
#
#   install(
#     FILES "${CMAKE_CURRENT_BINARY_DIR}/${FILE}"
#     DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
#   )
# endfunction()

# ##############################################################################
# WebP source files. Read the Makefile.am to get the source files.

# We expect the Makefiles to define the sources as defined in the first regex.
# E.g.: libimagedec_la_SOURCES  = image_dec.c image_dec.h
function(parse_Makefile_am FOLDER VAR SRC_REGEX)
  file(READ ${FOLDER}/Makefile.am MAKEFILE_AM)
  string(REGEX MATCHALL
               "${SRC_REGEX}_SOURCES[ ]*\\+?=[ ]+[0-9a-z\\._ ]*"
               FILES_PER_LINE
               ${MAKEFILE_AM})
  set(SRCS ${${VAR}})
  foreach(FILES ${FILES_PER_LINE})
    string(FIND ${FILES} "=" OFFSET)
    math(EXPR OFFSET "${OFFSET} + 2")
    string(SUBSTRING ${FILES}
                     ${OFFSET}
                     -1
                     FILES)
    if(FILES)
      string(REGEX MATCHALL
                   "[0-9a-z\\._]+"
                   FILES
                   ${FILES})
      foreach(FILE ${FILES})
        list(APPEND SRCS ${FOLDER}/${FILE})
      endforeach()
    endif()
  endforeach()
  set(${VAR} ${SRCS} PARENT_SCOPE)
endfunction()

function(add_webp_extern_definitions TARGET_LIB)
    if (BUILD_SHARED_LIBS AND WIN32)
        target_compile_definitions(${TARGET_LIB} PRIVATE "WEBP_EXTERN=__declspec(dllexport)")
        target_compile_definitions(${TARGET_LIB} INTERFACE "WEBP_EXTERN=__declspec(dllimport)")
    endif()
endfunction()

set(WEBP_SRC_DIR ${CMAKE_CURRENT_SOURCE_DIR}/src)
parse_makefile_am(${WEBP_SRC_DIR}/dec "WEBP_DEC_SRCS" "")
parse_makefile_am(${WEBP_SRC_DIR}/demux "WEBP_DEMUX_SRCS" "")
parse_makefile_am(${WEBP_SRC_DIR}/dsp "WEBP_DSP_COMMON_SRCS" "COMMON")
parse_makefile_am(${WEBP_SRC_DIR}/dsp "WEBP_DSP_ENC_SRCS" "ENC")
parse_makefile_am(${WEBP_SRC_DIR}/dsp "WEBP_DSP_ENC_SRCS" "dsp_[^ ]*")
parse_makefile_am(${WEBP_SRC_DIR}/dsp "WEBP_DSP_DEC_SRCS" "decode_[^ ]*")
parse_makefile_am(${WEBP_SRC_DIR}/enc "WEBP_ENC_SRCS" "")
parse_makefile_am(${WEBP_SRC_DIR}/utils "WEBP_UTILS_COMMON_SRCS" "COMMON")
parse_makefile_am(${WEBP_SRC_DIR}/utils "WEBP_UTILS_ENC_SRCS" "ENC")
parse_makefile_am(${WEBP_SRC_DIR}/utils "WEBP_UTILS_DEC_SRCS" "decode_[^ ]*")

# Remove the files specific to SIMD we don't use.
foreach(FILE ${WEBP_SIMD_FILES_NOT_TO_INCLUDE})
  list(REMOVE_ITEM WEBP_DSP_ENC_SRCS ${FILE})
  list(REMOVE_ITEM WEBP_DSP_DEC_SRCS ${FILE})
endforeach()

# Generate the config.h file.
configure_file(${CMAKE_CURRENT_LIST_DIR}/cmake/config.h.in
               ${CMAKE_CURRENT_BINARY_DIR}/src/webp/config.h)
add_definitions(-DHAVE_CONFIG_H)

# ##############################################################################
# Build the webpdecoder library.

# Creates a source file with an unused stub function in $CMAKE_BINARY_DIR and
# adds it to the specified target. Currently used only with Xcode.
#
# See also:
# https://cmake.org/cmake/help/v3.18/command/add_library.html#object-libraries
# "Some native build systems (such as Xcode) may not like targets that have
# only object files, so consider adding at least one real source file to any
# target that references $<TARGET_OBJECTS:objlib>."
function(libwebp_add_stub_file TARGET)
  set(stub_source_dir "${CMAKE_BINARY_DIR}")
  set(stub_source_file
      "${stub_source_dir}/libwebp_${TARGET}_stub.c")
  set(stub_source_code
      "// Generated file. DO NOT EDIT!\n"
      "// C source file created for target ${TARGET}.\n"
      "void libwebp_${TARGET}_stub_function(void)\;\n"
      "void libwebp_${TARGET}_stub_function(void) {}\n")
  file(WRITE "${stub_source_file}" ${stub_source_code})

  target_sources(${TARGET} PRIVATE ${stub_source_file})
endfunction()

if(MSVC)
  # avoid security warnings for e.g., fopen() used in the examples.
  add_definitions(-D_CRT_SECURE_NO_WARNINGS)
else()
  add_definitions(-Wall)
endif()
include_directories(${WEBP_DEP_INCLUDE_DIRS})
add_library(webpdecode OBJECT ${WEBP_DEC_SRCS})
target_include_directories(webpdecode
                           PRIVATE ${CMAKE_CURRENT_BINARY_DIR}
                                   ${CMAKE_CURRENT_SOURCE_DIR})
add_webp_extern_definitions(webpdecode)

add_library(webpdspdecode OBJECT ${WEBP_DSP_COMMON_SRCS} ${WEBP_DSP_DEC_SRCS})
target_include_directories(webpdspdecode
                           PRIVATE ${CMAKE_CURRENT_BINARY_DIR}
                                   ${CMAKE_CURRENT_SOURCE_DIR})
add_library(webputilsdecode
            OBJECT
            ${WEBP_UTILS_COMMON_SRCS}
            ${WEBP_UTILS_DEC_SRCS})
target_include_directories(webputilsdecode
                           PRIVATE ${CMAKE_CURRENT_BINARY_DIR}
                                   ${CMAKE_CURRENT_SOURCE_DIR})

set(PUBLIC_HEADERS_WDC ${CMAKE_CURRENT_SOURCE_DIR}/src/webp/decode.h ${CMAKE_CURRENT_SOURCE_DIR}/src/webp/types.h)
if(WIN32)
   set(PUBLIC_HEADERS_WDC ${PUBLIC_HEADERS_WDC} ${CMAKE_CURRENT_SOURCE_DIR}/src/libwebpdecoder.rc)
endif()
add_library(webpdecoder ${LIB_TYPE} ${PUBLIC_HEADERS_WDC}
    $<TARGET_OBJECTS:webpdecode>
    $<TARGET_OBJECTS:webpdspdecode>
    $<TARGET_OBJECTS:webputilsdecode>
)

add_webp_extern_definitions(webpdecoder)

if(XCODE)
  libwebp_add_stub_file(webpdecoder)
endif()

target_link_libraries(webpdecoder ${WEBP_DEP_LIBRARIES})

target_include_directories(
  webpdecoder
  PRIVATE ${CMAKE_CURRENT_BINARY_DIR} ${CMAKE_CURRENT_SOURCE_DIR}
  INTERFACE $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
            # $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
            )

if(OSX_FRAMEWORK)
    set_property(SOURCE ${PUBLIC_HEADERS_WDC}
        PROPERTY MACOSX_PACKAGE_LOCATION "Headers/webp"
    )
    set_target_properties(webpdecoder PROPERTIES
        FRAMEWORK TRUE
        FRAMEWORK_VERSION ${FRAMEWORK_VERSION}
        MACOSX_FRAMEWORK_SHORT_VERSION_STRING ${VERSION}
        MACOSX_FRAMEWORK_BUNDLE_VERSION ${VERSION}
        MACOSX_FRAMEWORK_IDENTIFIER com.google.webpdecoder
        XCODE_ATTRIBUTE_INSTALL_PATH "@rpath"
        # PUBLIC_HEADER "${PUBLIC_HEADERS_WDC}"
    )
else()
    set_target_properties(
      webpdecoder
      PROPERTIES PUBLIC_HEADER "${PUBLIC_HEADERS_WDC}")
endif()

# configure_pkg_config("src/libwebpdecoder.pc")

# Build the webp library.
add_library(webpencode OBJECT ${WEBP_ENC_SRCS})
target_include_directories(webpencode
                           PRIVATE ${CMAKE_CURRENT_BINARY_DIR}
                                   ${CMAKE_CURRENT_SOURCE_DIR})
add_webp_extern_definitions(webpencode)

add_library(webpdsp
            OBJECT
            ${WEBP_DSP_COMMON_SRCS}
            ${WEBP_DSP_DEC_SRCS}
            ${WEBP_DSP_ENC_SRCS})

target_include_directories(webpdsp
                           PRIVATE ${CMAKE_CURRENT_BINARY_DIR}
                                   ${CMAKE_CURRENT_SOURCE_DIR})
add_webp_extern_definitions(webpdsp)

add_library(webputils
            OBJECT
            ${WEBP_UTILS_COMMON_SRCS}
            ${WEBP_UTILS_DEC_SRCS}
            ${WEBP_UTILS_ENC_SRCS})
target_include_directories(webputils
                           PRIVATE ${CMAKE_CURRENT_BINARY_DIR}
                                   ${CMAKE_CURRENT_SOURCE_DIR})
add_webp_extern_definitions(webputils)

set(PUBLIC_HEADERS_WP 
    ${CMAKE_CURRENT_SOURCE_DIR}/src/webp/decode.h
    ${CMAKE_CURRENT_SOURCE_DIR}/src/webp/encode.h
    ${CMAKE_CURRENT_SOURCE_DIR}/src/webp/types.h)

add_library(webp ${LIB_TYPE} ${PUBLIC_HEADERS_WP}
            $<TARGET_OBJECTS:webpdecode>
            $<TARGET_OBJECTS:webpdsp>
            $<TARGET_OBJECTS:webpencode>
            $<TARGET_OBJECTS:webputils>)
add_webp_extern_definitions(webp)

if(XCODE)
  libwebp_add_stub_file(webp)
endif()

target_link_libraries(webp ${WEBP_DEP_LIBRARIES})
target_include_directories(webp
                           PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
                                #   $<INSTALL_INTERFACE:include>
                           PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}
                                   ${CMAKE_CURRENT_BINARY_DIR})



if(OSX_FRAMEWORK)
    set_property(SOURCE ${PUBLIC_HEADERS_WP}
        PROPERTY MACOSX_PACKAGE_LOCATION "Headers/webp"
    )
    set_target_properties(webp PROPERTIES
        FRAMEWORK TRUE
        FRAMEWORK_VERSION ${FRAMEWORK_VERSION}
        MACOSX_FRAMEWORK_SHORT_VERSION_STRING ${VERSION}
        MACOSX_FRAMEWORK_BUNDLE_VERSION ${VERSION}
        MACOSX_FRAMEWORK_IDENTIFIER com.google.webp
        XCODE_ATTRIBUTE_INSTALL_PATH "@rpath"
        # PUBLIC_HEADER "${PUBLIC_HEADERS_WP}"
    )
else()
    set_target_properties(
      webp
      PROPERTIES PUBLIC_HEADER "${PUBLIC_HEADERS_WP}")
endif()

# Make sure the OBJECT libraries are built with position independent code (it is
# not ON by default).
set_target_properties(webpdecode
                      webpdspdecode
                      webputilsdecode
                      webpencode
                      webpdsp
                      webputils
                      PROPERTIES POSITION_INDEPENDENT_CODE ON)
# configure_pkg_config("src/libwebp.pc")

set(PUBLIC_HEADERS_WDU
${CMAKE_CURRENT_SOURCE_DIR}/src/webp/decode.h
${CMAKE_CURRENT_SOURCE_DIR}/src/webp/demux.h
${CMAKE_CURRENT_SOURCE_DIR}/src/webp/mux_types.h
${CMAKE_CURRENT_SOURCE_DIR}/src/webp/types.h
)

add_library(webpdemux ${LIB_TYPE} ${WEBP_DEMUX_SRCS} ${PUBLIC_HEADERS_WDU})
target_link_libraries(webpdemux webp)
target_include_directories(webpdemux
                           PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}
                                   ${CMAKE_CURRENT_BINARY_DIR}
                           PUBLIC $<INSTALL_INTERFACE:include>)
add_webp_extern_definitions(webpdemux)

if(OSX_FRAMEWORK)
    set_property(SOURCE ${PUBLIC_HEADERS_WDU}
        PROPERTY MACOSX_PACKAGE_LOCATION "Headers/webp"
    )
    set_target_properties(webpdemux PROPERTIES
        FRAMEWORK TRUE
        FRAMEWORK_VERSION ${FRAMEWORK_VERSION}
        MACOSX_FRAMEWORK_SHORT_VERSION_STRING ${VERSION}
        MACOSX_FRAMEWORK_BUNDLE_VERSION ${VERSION}
        MACOSX_FRAMEWORK_IDENTIFIER com.google.webpdemux
        XCODE_ATTRIBUTE_INSTALL_PATH "@rpath"
        # PUBLIC_HEADER "${PUBLIC_HEADERS_WDU}"
    )
else()
    set_target_properties(
      webpdemux
      PROPERTIES PUBLIC_HEADER "${PUBLIC_HEADERS_WDU}")
endif()

# configure_pkg_config("src/demux/libwebpdemux.pc")

# Set the version numbers.
macro(set_version FILE TARGET_NAME NAME_IN_MAKEFILE)
  file(READ ${CMAKE_CURRENT_SOURCE_DIR}/src/${FILE} SOURCE_FILE)
  string(REGEX MATCH
               "${NAME_IN_MAKEFILE}_la_LDFLAGS[^\n]* -version-info [0-9:]+"
               TMP
               ${SOURCE_FILE})
  string(REGEX MATCH
               "[0-9:]+"
               TMP
               ${TMP})
  string(REGEX
         REPLACE ":"
                 " "
                 LT_VERSION
                 ${TMP})
  # See the libtool docs for more information:
  # https://www.gnu.org/software/libtool/manual/libtool.html#Updating-version-info
  #
  # c=<current>, a=<age>, r=<revision>
  #
  # libtool generates a .so file as .so.[c-a].a.r, while -version-info c:r:a is
  # passed to libtool.
  #
  # We set FULL = [c-a].a.r and MAJOR = [c-a].
  separate_arguments(LT_VERSION)
  list(GET LT_VERSION 0 LT_CURRENT)
  list(GET LT_VERSION 1 LT_REVISION)
  list(GET LT_VERSION 2 LT_AGE)
  math(EXPR LT_CURRENT_MINUS_AGE "${LT_CURRENT} - ${LT_AGE}")

  set_target_properties(
    ${TARGET_NAME}
    PROPERTIES VERSION
               ${LT_CURRENT_MINUS_AGE}.${LT_AGE}.${LT_REVISION}
               SOVERSION
               ${LT_CURRENT_MINUS_AGE})
endmacro()

set_version(Makefile.am webp webp)
set_version(Makefile.am webpdecoder webpdecoder)
set_version(demux/Makefile.am webpdemux webpdemux)

# Define the libraries to install.
list(APPEND INSTALLED_LIBRARIES webpdecoder webp webpdemux)

# Deal with SIMD. Change the compile flags for SIMD files we use.
list(LENGTH WEBP_SIMD_FILES_TO_INCLUDE WEBP_SIMD_FILES_TO_INCLUDE_LENGTH)
math(EXPR WEBP_SIMD_FILES_TO_INCLUDE_RANGE
     "${WEBP_SIMD_FILES_TO_INCLUDE_LENGTH}-1")

foreach(I_FILE RANGE ${WEBP_SIMD_FILES_TO_INCLUDE_RANGE})
  list(GET WEBP_SIMD_FILES_TO_INCLUDE ${I_FILE} FILE)
  list(GET WEBP_SIMD_FLAGS_TO_INCLUDE ${I_FILE} SIMD_COMPILE_FLAG)
  set_source_files_properties(${FILE}
                              PROPERTIES
                              COMPILE_FLAGS
                              ${SIMD_COMPILE_FLAG})
endforeach()

if(NOT WEBP_BUILD_LIBWEBPMUX)
  set(WEBP_BUILD_GIF2WEBP OFF)
  set(WEBP_BUILD_IMG2WEBP OFF)
  set(WEBP_BUILD_WEBPMUX OFF)
endif()

if(WEBP_BUILD_GIF2WEBP AND NOT GIF_FOUND)
  set(WEBP_BUILD_GIF2WEBP OFF)
endif()

if(WEBP_BUILD_ANIM_UTILS AND NOT GIF_FOUND)
  set(WEBP_BUILD_ANIM_UTILS OFF)
endif()

# Build the executables if asked for.
if(WEBP_BUILD_ANIM_UTILS
   OR WEBP_BUILD_CWEBP
   OR WEBP_BUILD_DWEBP
   OR WEBP_BUILD_GIF2WEBP
   OR WEBP_BUILD_IMG2WEBP
   OR WEBP_BUILD_VWEBP)

  # Example utility library.
  parse_makefile_am(${CMAKE_CURRENT_SOURCE_DIR}/examples "EXAMPLEUTIL_SRCS"
                    "example_util_[^ ]*")
  list(APPEND EXAMPLEUTIL_SRCS ${CMAKE_CURRENT_SOURCE_DIR}/examples/stopwatch.h)
  add_library(exampleutil STATIC ${EXAMPLEUTIL_SRCS})
  target_link_libraries(exampleutil imageioutil)
  target_include_directories(
    exampleutil
    PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>)

  parse_makefile_am(${CMAKE_CURRENT_SOURCE_DIR}/imageio "IMAGEIOUTILS_SRCS"
                    "imageio_util_[^ ]*")
  add_library(imageioutil STATIC ${IMAGEIOUTILS_SRCS})
  target_link_libraries(imageioutil webp)

  # Image-decoding utility library.
  parse_makefile_am(${CMAKE_CURRENT_SOURCE_DIR}/imageio "IMAGEDEC_SRCS"
                    "imagedec_[^ ]*")
  add_library(imagedec STATIC ${IMAGEDEC_SRCS})
  target_link_libraries(imagedec
                        imageioutil
                        webpdemux
                        webp
                        ${WEBP_DEP_IMG_LIBRARIES})

  # Image-encoding utility library.
  parse_makefile_am(${CMAKE_CURRENT_SOURCE_DIR}/imageio "IMAGEENC_SRCS"
                    "imageenc_[^ ]*")
  add_library(imageenc STATIC ${IMAGEENC_SRCS})
  target_link_libraries(imageenc imageioutil webp)

  set_property(TARGET exampleutil
                      imageioutil
                      imagedec
                      imageenc
               PROPERTY INCLUDE_DIRECTORIES ${CMAKE_CURRENT_SOURCE_DIR}/src
                        ${CMAKE_CURRENT_BINARY_DIR}/src)
endif()

if(WEBP_BUILD_DWEBP)
  # dwebp
  parse_makefile_am(${CMAKE_CURRENT_SOURCE_DIR}/examples "DWEBP_SRCS" "dwebp")
  add_executable(dwebp ${DWEBP_SRCS})
  target_link_libraries(dwebp exampleutil imagedec imageenc)
  target_include_directories(dwebp PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/src)
  install(TARGETS dwebp RUNTIME DESTINATION ${INSTALL_BIN_DIR})
endif()

if(WEBP_BUILD_CWEBP)
  # cwebp
  parse_makefile_am(${CMAKE_CURRENT_SOURCE_DIR}/examples "CWEBP_SRCS" "cwebp")
  add_executable(cwebp ${CWEBP_SRCS})
  target_link_libraries(cwebp exampleutil imagedec webp)
  target_include_directories(cwebp PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/src)
  install(TARGETS cwebp RUNTIME DESTINATION ${INSTALL_BIN_DIR})
endif()

if(WEBP_BUILD_LIBWEBPMUX)

  set(PUBLIC_HEADERS_WM
    ${CMAKE_CURRENT_SOURCE_DIR}/src/webp/mux.h
    ${CMAKE_CURRENT_SOURCE_DIR}/src/webp/mux_types.h
    ${CMAKE_CURRENT_SOURCE_DIR}/src/webp/types.h
  )

  parse_makefile_am(${CMAKE_CURRENT_SOURCE_DIR}/src/mux "WEBP_MUX_SRCS" "")
  add_library(libwebpmux ${WEBP_MUX_SRCS} ${PUBLIC_HEADERS_WM})
  target_link_libraries(libwebpmux webp)
  target_include_directories(libwebpmux
                             PRIVATE ${CMAKE_CURRENT_BINARY_DIR}
                                     ${CMAKE_CURRENT_SOURCE_DIR})
  add_webp_extern_definitions(libwebpmux)
  set_version(mux/Makefile.am libwebpmux webpmux)
  set_target_properties(libwebpmux PROPERTIES OUTPUT_NAME webpmux)
  list(APPEND INSTALLED_LIBRARIES libwebpmux)
  #configure_pkg_config("src/mux/libwebpmux.pc")


  if(OSX_FRAMEWORK)
    set_property(SOURCE ${PUBLIC_HEADERS_WM}
        PROPERTY MACOSX_PACKAGE_LOCATION "Headers/webp"
    )
    set_target_properties(libwebpmux PROPERTIES
        FRAMEWORK TRUE
        FRAMEWORK_VERSION ${FRAMEWORK_VERSION}
        MACOSX_FRAMEWORK_SHORT_VERSION_STRING ${VERSION}
        MACOSX_FRAMEWORK_BUNDLE_VERSION ${VERSION}
        MACOSX_FRAMEWORK_IDENTIFIER com.google.webpmux
        XCODE_ATTRIBUTE_INSTALL_PATH "@rpath"
        # PUBLIC_HEADER "${PUBLIC_HEADERS_WM}"
    )
  else()
      set_target_properties(libwebpmux PROPERTIES PUBLIC_HEADER
        "${PUBLIC_HEADERS_WM}"
        DEFINE_SYMBOL WEBP_EXPORTS)
  endif()
endif()

if(WEBP_BUILD_GIF2WEBP)
  # gif2webp
  include_directories(${WEBP_DEP_GIF_INCLUDE_DIRS})
  parse_makefile_am(${CMAKE_CURRENT_SOURCE_DIR}/examples "GIF2WEBP_SRCS"
                    "gif2webp")
  add_executable(gif2webp ${GIF2WEBP_SRCS})
  target_link_libraries(gif2webp
                        exampleutil
                        imageioutil
                        webp
                        libwebpmux
                        ${WEBP_DEP_GIF_LIBRARIES})
  target_include_directories(gif2webp PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/src)
  install(TARGETS gif2webp RUNTIME DESTINATION ${INSTALL_BIN_DIR})
endif()

if(WEBP_BUILD_IMG2WEBP)
  # img2webp
  include_directories(${WEBP_DEP_IMG_INCLUDE_DIRS})
  parse_makefile_am(${CMAKE_CURRENT_SOURCE_DIR}/examples "IMG2WEBP_SRCS"
                    "img2webp")
  add_executable(img2webp ${IMG2WEBP_SRCS})
  target_link_libraries(img2webp
                        exampleutil
                        imagedec
                        imageioutil
                        webp
                        libwebpmux)
  target_include_directories(img2webp PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/src)
  install(TARGETS img2webp RUNTIME DESTINATION ${INSTALL_BIN_DIR})
endif()

if(WEBP_BUILD_VWEBP)
  # vwebp
  find_package(GLUT)
  if(GLUT_FOUND)
    include_directories(${WEBP_DEP_IMG_INCLUDE_DIRS})
    parse_makefile_am(${CMAKE_CURRENT_SOURCE_DIR}/examples "VWEBP_SRCS" "vwebp")
    add_executable(vwebp ${VWEBP_SRCS})
    target_link_libraries(vwebp
                          ${OPENGL_LIBRARIES}
                          exampleutil
                          GLUT::GLUT
                          imageioutil
                          webp
                          webpdemux)
    target_include_directories(vwebp
                               PRIVATE GLUT::GLUT
                                       ${CMAKE_CURRENT_BINARY_DIR}/src
                                       ${OPENGL_INCLUDE_DIR})
    install(TARGETS vwebp RUNTIME DESTINATION ${INSTALL_BIN_DIR})
    if(${CMAKE_SYSTEM_NAME} STREQUAL "Darwin")
      check_c_compiler_flag("-Wno-deprecated-declarations" HAS_NO_DEPRECATED)
      if(HAS_NO_DEPRECATED)
        target_compile_options(vwebp PRIVATE "-Wno-deprecated-declarations")
      endif()
    endif()
  endif()
endif()

if(WEBP_BUILD_WEBPINFO)
  # webpinfo
  include_directories(${WEBP_DEP_IMG_INCLUDE_DIRS})
  parse_makefile_am(${CMAKE_CURRENT_SOURCE_DIR}/examples "WEBPINFO_SRCS"
                    "webpinfo")
  add_executable(webpinfo ${WEBPINFO_SRCS})
  target_link_libraries(webpinfo exampleutil imageioutil)
  target_include_directories(webpinfo PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/src)
  install(TARGETS webpinfo RUNTIME DESTINATION ${INSTALL_BIN_DIR})
endif()

if(WEBP_BUILD_WEBPMUX)
  # webpmux
  parse_makefile_am(${CMAKE_CURRENT_SOURCE_DIR}/examples "WEBPMUX_SRCS"
                    "webpmux")
  add_executable(webpmux ${WEBPMUX_SRCS})
  target_link_libraries(webpmux exampleutil imageioutil libwebpmux webp)
  target_include_directories(webpmux PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/src)
  install(TARGETS webpmux RUNTIME DESTINATION ${INSTALL_BIN_DIR})
endif()

if(WEBP_BUILD_EXTRAS)
  set(EXTRAS_MAKEFILE "${CMAKE_CURRENT_SOURCE_DIR}/extras")
  parse_makefile_am(${EXTRAS_MAKEFILE} "WEBP_EXTRAS_SRCS" "libwebpextras_la")
  parse_makefile_am(${EXTRAS_MAKEFILE} "GET_DISTO_SRCS" "get_disto")
  parse_makefile_am(${EXTRAS_MAKEFILE} "WEBP_QUALITY_SRCS" "webp_quality")
  parse_makefile_am(${EXTRAS_MAKEFILE} "VWEBP_SDL_SRCS" "vwebp_sdl")

  # get_disto
  add_executable(get_disto ${GET_DISTO_SRCS})
  target_link_libraries(get_disto imagedec)
  target_include_directories(get_disto
                             PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}
                                     ${CMAKE_CURRENT_SOURCE_DIR}/src
                                     ${CMAKE_CURRENT_BINARY_DIR}/src)
  install(TARGETS get_disto RUNTIME DESTINATION ${INSTALL_BIN_DIR})

  # webp_quality
  add_executable(webp_quality ${WEBP_QUALITY_SRCS} ${WEBP_EXTRAS_SRCS})
  target_link_libraries(webp_quality exampleutil imagedec webpdsp)
  target_include_directories(webp_quality
                             PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}
                                     ${CMAKE_CURRENT_BINARY_DIR})
  install(TARGETS webp_quality RUNTIME DESTINATION ${INSTALL_BIN_DIR})

  # vwebp_sdl
  find_package(SDL)
  if(SDL_FOUND)
    add_executable(vwebp_sdl ${VWEBP_SDL_SRCS})
    target_link_libraries(vwebp_sdl ${SDL_LIBRARY} imageioutil webp)
    target_include_directories(vwebp_sdl
                               PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}
                                       ${CMAKE_CURRENT_SOURCE_DIR}/src
                                       ${CMAKE_CURRENT_BINARY_DIR}
                                       ${CMAKE_CURRENT_BINARY_DIR}/src
                                       ${SDL_INCLUDE_DIR})
    set(WEBP_HAVE_SDL 1)
    target_compile_definitions(vwebp_sdl PUBLIC WEBP_HAVE_SDL)
    install(TARGETS vwebp_sdl RUNTIME DESTINATION ${INSTALL_BIN_DIR})
  endif()
endif()

if(WEBP_BUILD_WEBP_JS)
  # wasm2js does not support SIMD.
  if(NOT WEBP_ENABLE_SIMD)
    # JavaScript version
    add_executable(webp_js ${CMAKE_CURRENT_SOURCE_DIR}/extras/webp_to_sdl.c)
    target_link_libraries(webp_js webpdecoder SDL)
    target_include_directories(webp_js PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
    set(WEBP_HAVE_SDL 1)
    set_target_properties(
      webp_js
      PROPERTIES LINK_FLAGS "-s WASM=0 \
         -s EXPORTED_FUNCTIONS='[\"_WebpToSDL\"]' -s INVOKE_RUN=0 \
         -s EXPORTED_RUNTIME_METHODS='[\"cwrap\"]'")
    set_target_properties(webp_js PROPERTIES OUTPUT_NAME webp)
    target_compile_definitions(webp_js PUBLIC EMSCRIPTEN WEBP_HAVE_SDL)
  endif()

  # WASM version
  add_executable(webp_wasm ${CMAKE_CURRENT_SOURCE_DIR}/extras/webp_to_sdl.c)
  target_link_libraries(webp_wasm webpdecoder SDL)
  target_include_directories(webp_wasm PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
  set_target_properties(
    webp_wasm
    PROPERTIES LINK_FLAGS "-s WASM=1 -s 'BINARYEN_METHOD=\"native-wasm\"' \
       -s EXPORTED_FUNCTIONS='[\"_WebpToSDL\"]' -s INVOKE_RUN=0 \
       -s EXTRA_EXPORTED_RUNTIME_METHODS='[\"cwrap\"]'")
  target_compile_definitions(webp_wasm PUBLIC EMSCRIPTEN WEBP_HAVE_SDL)

  target_compile_definitions(webpdspdecode PUBLIC EMSCRIPTEN)
endif()

if(WEBP_BUILD_ANIM_UTILS)
  # anim_diff
  include_directories(${WEBP_DEP_IMG_INCLUDE_DIRS} ${WEBP_DEP_GIF_INCLUDE_DIRS})
  parse_makefile_am(${CMAKE_CURRENT_SOURCE_DIR}/examples "ANIM_DIFF_SRCS"
                    "anim_diff")
  add_executable(anim_diff ${ANIM_DIFF_SRCS})
  target_link_libraries(anim_diff
                        exampleutil
                        imagedec
                        imageenc
                        imageioutil
                        webp
                        libwebpmux
                        webpdemux
                        ${WEBP_DEP_GIF_LIBRARIES})
  target_include_directories(anim_diff PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/src)

  # anim_dump
  include_directories(${WEBP_DEP_IMG_INCLUDE_DIRS} ${WEBP_DEP_GIF_INCLUDE_DIRS})
  parse_makefile_am(${CMAKE_CURRENT_SOURCE_DIR}/examples "ANIM_DUMP_SRCS"
                    "anim_dump")
  add_executable(anim_dump ${ANIM_DUMP_SRCS})
  target_link_libraries(anim_dump
                        exampleutil
                        imagedec
                        imageenc
                        imageioutil
                        webp
                        libwebpmux
                        webpdemux
                        ${WEBP_DEP_GIF_LIBRARIES})
  target_include_directories(anim_dump PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/src)
endif()

foreach(INSTALLED_LIBRARY ${INSTALLED_LIBRARIES})
    set(PACKAGE_NAME ${INSTALLED_LIBRARY})
    if(PACKAGE_NAME STREQUAL "libwebpmux")
        set(PACKAGE_NAME_FIX webpmux)
    else()
        set(PACKAGE_NAME_FIX ${PACKAGE_NAME})
    endif()
    string(TOUPPER ${PACKAGE_NAME_FIX} PACKAGE_UPPER_NAME)

    if(OSX_FRAMEWORK)
        set(INSTALL_INC_DIR "${INSTALL_LIB_DIR}/${PACKAGE_NAME_FIX}.framework/Headers")
    else()
        set(INSTALL_INC_DIR ${CMAKE_INSTALL_INCLUDEDIR})
    endif()

    # Install the different headers and libraries.
    # include(GNUInstallDirs)
    install(TARGETS ${INSTALLED_LIBRARY}
            EXPORT ${PACKAGE_UPPER_NAME}Targets
            PUBLIC_HEADER DESTINATION ${INSTALL_INC_DIR}/webp
            INCLUDES
            DESTINATION ${INSTALL_INC_DIR}
            ARCHIVE DESTINATION ${INSTALL_LIB_DIR}
            LIBRARY DESTINATION ${INSTALL_LIB_DIR}
            RUNTIME DESTINATION ${INSTALL_BIN_DIR}
            FRAMEWORK DESTINATION ${INSTALL_LIB_DIR})

    # Add all targets to the build-tree export set
    export(TARGETS ${INSTALLED_LIBRARY}
        FILE ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Targets.cmake)

    # Create the <Package>Config.cmake file
    configure_file(${PROJECT_SOURCE_DIR}/cmake/PackageConfig.cmake.in
        ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake @ONLY)

    if(OSX_FRAMEWORK)
        set(INSTALL_CMAKECONF_DIR ${INSTALL_LIB_DIR}/${PACKAGE_NAME_FIX}.framework/Resources/CMake CACHE INTERNAL "Installation directory for cmake config files" FORCE)
    else()
        set(INSTALL_CMAKECONF_DIR ${CMAKE_INSTALL_DATADIR}/${PACKAGE_NAME_FIX}/CMake CACHE INTERNAL "Installation directory for cmake config files" FORCE)
    endif()

    if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL)
        # Install the <Package>Config.cmake
        install(FILES
          ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake
          DESTINATION ${INSTALL_CMAKECONF_DIR} COMPONENT dev)

        # Install the export set for use with the install-tree
        install(EXPORT ${PACKAGE_UPPER_NAME}Targets DESTINATION ${INSTALL_CMAKECONF_DIR} COMPONENT dev)
    endif()

    # Export package ===============================================================

    if(REGISTER_PACKAGE)
        # Export the package for use from the build-tree
        # (this registers the build-tree with a global CMake-registry)
        export(PACKAGE ${PACKAGE_UPPER_NAME})
    endif()

endforeach()
# set(ConfigPackageLocation ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/cmake/)
# install(EXPORT ${PROJECT_NAME}Targets
#         NAMESPACE ${PROJECT_NAME}::
#         DESTINATION ${ConfigPackageLocation})
#
# # Create the CMake version file.
# include(CMakePackageConfigHelpers)
# write_basic_package_version_file(
#   "${CMAKE_CURRENT_BINARY_DIR}/WebPConfigVersion.cmake"
#   VERSION ${PACKAGE_VERSION}
#   COMPATIBILITY AnyNewerVersion)
#
# # Create the Config file.
# include(CMakePackageConfigHelpers)
# configure_package_config_file(
#   ${CMAKE_CURRENT_SOURCE_DIR}/cmake/WebPConfig.cmake.in
#   ${CMAKE_CURRENT_BINARY_DIR}/WebPConfig.cmake
#   INSTALL_DESTINATION
#   ${ConfigPackageLocation})
#
# # Install the generated CMake files.
# install(FILES "${CMAKE_CURRENT_BINARY_DIR}/WebPConfigVersion.cmake"
#               "${CMAKE_CURRENT_BINARY_DIR}/WebPConfig.cmake"
#         DESTINATION ${ConfigPackageLocation})

# Install the man pages.
set(MAN_PAGES
    cwebp.1
    dwebp.1
    gif2webp.1
    img2webp.1
    vwebp.1
    webpmux.1
    webpinfo.1)
set(EXEC_BUILDS
    "CWEBP"
    "DWEBP"
    "GIF2WEBP"
    "IMG2WEBP"
    "VWEBP"
    "WEBPMUX"
    "WEBPINFO")
list(LENGTH MAN_PAGES MAN_PAGES_LENGTH)
math(EXPR MAN_PAGES_RANGE "${MAN_PAGES_LENGTH} - 1")

foreach(I_MAN RANGE ${MAN_PAGES_RANGE})
  list(GET EXEC_BUILDS ${I_MAN} EXEC_BUILD)
  if(WEBP_BUILD_${EXEC_BUILD})
    list(GET MAN_PAGES ${I_MAN} MAN_PAGE)
    install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/man/${MAN_PAGE}
            DESTINATION ${INSTALL_DOC_DIR}/man1
            COMPONENT doc)
  endif()
endforeach()

# ==============================================================================

add_custom_target(uninstall COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)

# Archiving ====================================================================

set(CPACK_PACKAGE_NAME "${PROJECT_NAME}")
set(CPACK_PACKAGE_VENDOR "${PACKAGE_VENDOR}")
set(CPACK_PACKAGE_VERSION "${VERSION}")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "${PROJECT_NAME} Installation")
set(CPACK_PACKAGE_RELOCATABLE TRUE)
set(CPACK_ARCHIVE_COMPONENT_INSTALL ON)
set(CPACK_GENERATOR "ZIP")
set(CPACK_MONOLITHIC_INSTALL ON)
set(CPACK_STRIP_FILES TRUE)

# Get cpack zip archive name
get_cpack_filename(${VERSION} PROJECT_CPACK_FILENAME)
set(CPACK_PACKAGE_FILE_NAME ${PROJECT_CPACK_FILENAME})

include(CPack)
