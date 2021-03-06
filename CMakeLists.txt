cmake_minimum_required (VERSION 2.6)

project (emf2svg)

# Project version (sematic versionning)
set(EMFSVG_VERSION_MAJOR 0)
set(EMFSVG_VERSION_MINOR 0)
set(EMFSVG_VERSION_PATCH 1)

set(EMFSVG_VERSION ${EMFSVG_VERSION_MAJOR}.${EMFSVG_VERSION_MINOR}.${EMFSVG_VERSION_PATCH})

set_directory_properties(PROPERTIES ADDITIONAL_MAKE_CLEAN_FILES external-build)

# Options
option(DEBUG     "compile with debug symbol"    OFF)
option(STATIC    "compile statically"           OFF)
option(USE_CLANG "build application with clang" OFF)
option(USE_GCC   "build application with gcc"   OFF)

if(USE_CLANG)
    set(CMAKE_CXX_COMPILER "clang++")
    set(CMAKE_CC_COMPILER  "clang"  )
endif(USE_CLANG)

if(USE_GCC)
    set(CMAKE_CXX_COMPILER "g++")
    set(CMAKE_CC_COMPILER  "gcc")
endif(USE_GCC)

if(DEBUG)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O0 -g")
    set(CMAKE_BUILD_TYPE Debug)
endif(DEBUG)

if(STATIC)
    set(SHARED "")
    set(CMAKE_FIND_LIBRARY_SUFFIXES ".a")
    set(BUILD_SHARED_LIBRARIES OFF)
    set(CMAKE_EXE_LINKER_FLAGS "-static")
else(STATIC)
    IF(${CMAKE_SYSTEM_NAME}    MATCHES "Darwin")
        set(SHARED "")
    ELSE(${CMAKE_SYSTEM_NAME}  MATCHES "Darwin")
        set(SHARED "SHARED")
    ENDIF(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
endif(STATIC)

if(UNIX)
  link_libraries(m)
endif(UNIX)

include(TestBigEndian)
TEST_BIG_ENDIAN(BIGENDIAN)
IF(${BIGENDIAN})
    add_definitions(-DWORDS_BIGENDIAN)
ENDIF(${BIGENDIAN})

# Build external dependancies if we are on OSX
IF(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
   # Mac OS X specific code
   set(MEMSTREAMLIB "memstream")
   add_definitions(-DDARWIN)

   include(ExternalProject)

   set(LIBCLIPPERBUILDDIR "${CMAKE_CURRENT_SOURCE_DIR}/external-build/clipper/")
   set(LIBCLIPPERHEADERDIR "${CMAKE_CURRENT_SOURCE_DIR}/deps/clipper/")
   set(LIBCLIPPERLINKDIR  "${LIBCLIPPERBUILDDIR}/src/clipper-standalone-build/lib/")
   set(LIBCLIPPERLIB  "clipper")

   ExternalProject_Add( clipper-standalone
        PREFIX ${LIBCLIPPERBUILDDIR}
        DOWNLOAD_DIR ${LIBCLIPPERBUILDDIR}/sources/
        DOWNLOAD_COMMAND cp -r ${CMAKE_CURRENT_SOURCE_DIR}/deps/libpolyclipping/ ./
        SOURCE_DIR ${LIBCLIPPERBUILDDIR}/sources/
        CONFIGURE_COMMAND cmake ${LIBCLIPPERBUILDDIR}/sources/cpp/
        BUILD_COMMAND ${MAKE}
   )

   set(ARGPHEADERDIR "${CMAKE_CURRENT_SOURCE_DIR}/deps/argp-standalone/")
   set(ARGPBUILDDIR "${CMAKE_CURRENT_SOURCE_DIR}/external-build/argp/")
   set(ARGPLINKDIR  "${ARGPBUILDDIR}/src/argp-standalone-build/")
   set(ARGPLIB  "argp")

   ExternalProject_Add( argp-standalone
        PREFIX ${ARGPBUILDDIR}
        DOWNLOAD_DIR ${ARGPBUILDDIR}/sources/
        DOWNLOAD_COMMAND cp -r ${CMAKE_CURRENT_SOURCE_DIR}/deps/argp-standalone/ ${ARGPBUILDDIR}/sources/
        SOURCE_DIR ${ARGPBUILDDIR}/sources/
        PATCH_COMMAND patch -b argp-fmtstream.h < ${CMAKE_CURRENT_SOURCE_DIR}/deps/patch/patch-argp-fmtstream.h
        CONFIGURE_COMMAND ${ARGPBUILDDIR}/sources/configure
        BUILD_COMMAND ${MAKE}
   )

   set(LIBICONVBUILDDIR "${CMAKE_CURRENT_SOURCE_DIR}/external-build/libiconv/")
   set(LIBICONVHEADERDIR "${CMAKE_CURRENT_SOURCE_DIR}/deps/libiconv/")
   set(LIBICONVLINKDIR  "${LIBICONVBUILDDIR}/src/iconv-standalone-build/lib/")
   set(LIBICONVLIB  "iconv")

   ExternalProject_Add( iconv-standalone
        PREFIX ${LIBICONVBUILDDIR}
        DOWNLOAD_DIR ${LIBICONVBUILDDIR}/sources/
        DOWNLOAD_COMMAND cp -r ${CMAKE_CURRENT_SOURCE_DIR}/deps/libiconv/ ./
        SOURCE_DIR ${LIBICONVBUILDDIR}/sources/
        CONFIGURE_COMMAND ${LIBICONVBUILDDIR}/sources/configure
        BUILD_COMMAND ${MAKE}
   )
ENDIF(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")

# linking directories
LINK_DIRECTORIES(
    ${CMAKE_BINARY_DIR}/
    /usr/local/lib
    /sw/lib/
    /usr/lib/
    ${ARGPLINKDIR}
    ${LIBICONVLINKDIR}
)

# headers directories
include_directories(
    ./inc/
    /usr/include/ 
    /sw/include/
    ${ARGPHEADERDIR}
    ${LIBICONVHEADERDIR}
)

# Compile the library
add_library(EMFSVG ${SHARED} src/lib/EMFSVG.c src/lib/EMFSVG_print.c src/lib/PMFSVG.c src/lib/PMFSVG_print.c src/lib/uemf_utf.c src/lib/uemf_endian.c src/lib/uemf.c src/lib/upmf.c src/lib/clipperWrapper.cpp)
IF(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
   # Mac OS X specific code
   add_library(memstream ${SHARED} src/lib/memstream.c)
ENDIF(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
set_target_properties(EMFSVG PROPERTIES VERSION ${EMFSVG_VERSION} SOVERSION ${EMFSVG_VERSION_MAJOR})

# Compile the executable
add_executable(emf2svg src/conv/emf2svg.cpp)
target_link_libraries(emf2svg EMFSVG ${LIBICONVLIB} ${MEMSTREAMLIB} ${ARGPLIB} ) 

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wall")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=c11 -Wall")

# install binaries and library
INSTALL(TARGETS emf2svg EMFSVG ${MEMSTREAMLIB}
  RUNTIME DESTINATION bin
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib
)
# install header file
INSTALL(FILES inc/EMFSVG.h DESTINATION "include")

# dependancies if we are on OSX
IF(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
   add_dependencies(emf2svg clipper-standalone)
   add_dependencies(emf2svg argp-standalone)
   add_dependencies(emf2svg iconv-standalone)
ENDIF(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
