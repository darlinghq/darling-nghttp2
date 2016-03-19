# nghttp2 - HTTP/2 C Library
#
# Copyright (c) 2012, 2013, 2014, 2015 Tatsuhiro Tsujikawa
# Copyright (c) 2016 Peter Wu <peter@lekensteyn.nl>
#
# Permission is hereby granted, free of charge, to any person obtaining
# a copy of this software and associated documentation files (the
# "Software"), to deal in the Software without restriction, including
# without limitation the rights to use, copy, modify, merge, publish,
# distribute, sublicense, and/or sell copies of the Software, and to
# permit persons to whom the Software is furnished to do so, subject to
# the following conditions:
#
# The above copyright notice and this permission notice shall be
# included in all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
# EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
# MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
# NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE
# LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION
# OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION
# WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

cmake_minimum_required(VERSION 3.0)
# XXX using 1.8.90 instead of 1.9.0-DEV
project(nghttp2 VERSION 1.8.90)

# See versioning rule:
#  http://www.gnu.org/software/libtool/manual/html_node/Updating-version-info.html
set(LT_CURRENT  19)
set(LT_REVISION 0)
set(LT_AGE      5)

set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")
include(Version)

math(EXPR LT_SOVERSION "${LT_CURRENT} - ${LT_AGE}")
set(LT_VERSION "${LT_SOVERSION}.${LT_AGE}.${LT_REVISION}")
set(PACKAGE_VERSION     "${PROJECT_VERSION}")
HexVersion(PACKAGE_VERSION_NUM ${PROJECT_VERSION_MAJOR} ${PROJECT_VERSION_MINOR} ${PROJECT_VERSION_PATCH})

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  set(CMAKE_BUILD_TYPE RelWithDebInfo CACHE STRING "Choose the build type" FORCE)

  # Include "None" as option to disable any additional (optimization) flags,
  # relying on just CMAKE_C_FLAGS and CMAKE_CXX_FLAGS (which are empty by
  # default). These strings are presented in cmake-gui.
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS
    "None" "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()

include(GNUInstallDirs)

# For Python bindings and documentation
# (Must be called before PythonLibs for matching versions.)
find_package(PythonInterp)

# Auto-detection of features that can be toggled
find_package(OpenSSL 1.0.1)
find_package(Libev 4.11)
find_package(ZLIB 1.2.3)
if(OPENSSL_FOUND AND LIBEV_FOUND AND ZLIB_FOUND)
  set(ENABLE_APP_DEFAULT ON)
else()
  set(ENABLE_APP_DEFAULT OFF)
endif()
find_package(Jansson  2.5)
set(ENABLE_HPACK_TOOLS_DEFAULT ${JANSSON_FOUND})
# 2.0.8 is required because we use evconnlistener_set_error_cb()
find_package(Libevent 2.0.8 COMPONENTS libevent openssl)
set(ENABLE_EXAMPLES_DEFAULT ${LIBEVENT_OPENSSL_FOUND})
find_package(Cython)
find_package(PythonLibs)
if(CYTHON_FOUND AND PYTHONLIBS_FOUND)
  set(ENABLE_PYTHON_BINDINGS_DEFAULT ON)
else()
  set(ENABLE_PYTHON_BINDINGS_DEFAULT OFF)
endif()

find_package(LibXml2 2.7.7)
set(WITH_LIBXML2_DEFAULT    ${LIBXML2_FOUND})
find_package(Jemalloc)
set(WITH_JEMALLOC_DEFAULT   ${JEMALLOC_FOUND})
find_package(Spdylay 1.3.2)
set(WITH_SPDYLAY_DEFAULT    ${SPDYLAY_FOUND})

include(CMakeOptions.txt)

if(ENABLE_LIB_ONLY AND (ENABLE_APP OR ENABLE_HPACK_TOOLS OR ENABLE_EXAMPLES OR
  ENABLE_PYTHON_BINDINGS))
  # Remember when disabled options are disabled for later diagnostics.
  set(ENABLE_LIB_ONLY_DISABLED_OTHERS 1)
else()
  set(ENABLE_LIB_ONLY_DISABLED_OTHERS 0)
endif()
if(ENABLE_LIB_ONLY)
  set(ENABLE_APP            OFF)
  set(ENABLE_HPACK_TOOLS    OFF)
  set(ENABLE_EXAMPLES       OFF)
  set(ENABLE_PYTHON_BINDINGS OFF)
endif()

# Do not disable assertions based on CMAKE_BUILD_TYPE.
foreach(_build_type "Release" "MinSizeRel" "RelWithDebInfo")
  foreach(_lang C CXX)
    string(TOUPPER "CMAKE_${_lang}_FLAGS_${_build_type}" _var)
    string(REGEX REPLACE "(^| )[/-]D *NDEBUG($| )" "" ${_var} "${${_var}}")
  endforeach()
endforeach()

#
# If we're running GCC or clang define _U_ to be "__attribute__((unused))"
# so we can use _U_ to flag unused function parameters and not get warnings
# about them. Otherwise, define _U_ to be an empty string so that _U_ used
# to flag an unused function parameters will compile with other compilers.
#
# XXX - similar hints for other compilers?
#
if(CMAKE_C_COMPILER_ID MATCHES "GNU" OR CMAKE_C_COMPILER_ID MATCHES "Clang")
  set(HINT_UNUSED_PARAM   "__attribute__((unused))")
  set(HINT_NORETURN       "__attribute__((noreturn))")
else()
  set(HINT_UNUSED_PARAM)
  set(HINT_NORETURN)
endif()

include(ExtractValidFlags)
foreach(_cxx1x_flag -std=c++11 -std=c++0x)
  extract_valid_cxx_flags(_cxx1x_flag_supported ${_cxx1x_flag})
  if(_cxx1x_flag_supported)
    set(CXX1XCXXFLAGS ${_cxx1x_flag})
    break()
  endif()
endforeach()

include(CMakePushCheckState)
include(CheckCXXSourceCompiles)
cmake_push_check_state()
set(CMAKE_REQUIRED_DEFINITIONS "${CXX1XCXXFLAGS}")
# Check that std::future is available.
check_cxx_source_compiles("
#include <vector>
#include <future>
int main() { std::vector<std::future<int>> v; }" HAVE_STD_FUTURE)
# Check that std::map::emplace is available for g++-4.7.
check_cxx_source_compiles("
#include <map>
int main() { std::map<int, int>().emplace(1, 2); }" HAVE_STD_MAP_EMPLACE)
cmake_pop_check_state()


# Checks for libraries.
# Additional libraries required for programs under src directory.
set(APP_LIBRARIES)

if(ENABLE_PYTHON_BINDINGS)
  if(NOT (CYTHON_FOUND AND PYTHONLIBS_FOUND))
    message(FATAL_ERROR "python bindings were requested "
      "(ENABLE_PYTHON_BINDINGS=1) but dependencies are not met.")
  endif()
  if(NOT PYTHON_VERSION_STRING STREQUAL PYTHONLIBS_VERSION_STRING)
    message(FATAL_ERROR
      "Python executable and library must have the same version!"
      " Found Python ${PYTHON_VERSION_STRING} and"
      " PythonLibs ${PYTHONLIBS_VERSION_STRING}"
    )
  endif()
endif()

set(CMAKE_THREAD_PREFER_PTHREAD 1)
find_package(Threads)
if(CMAKE_USE_PTHREADS_INIT)
  list(APPEND APP_LIBRARIES ${CMAKE_THREAD_LIBS_INIT})
endif()
# XXX android and C++, is this still needed in cmake?
# case "$host" in
#   *android*)
#     android_build=yes
#     # android does not need -pthread, but needs followng 3 libs for C++
#     APPLDFLAGS="$APPLDFLAGS -lstdc++ -latomic -lsupc++"

# dl: openssl requires libdl when it is statically linked.
# XXX shouldn't ${CMAKE_DL_LIBS} be appended to OPENSSL_LIBRARIES instead of
# APP_LIBRARIES if it is really specific to OpenSSL?

find_package(CUnit 2.1)
enable_testing()
set(HAVE_CUNIT      ${CUNIT_FOUND})
if(HAVE_CUNIT)
  add_custom_target(check COMMAND ${CMAKE_CTEST_COMMAND})
endif()

# openssl (for src)
set(HAVE_OPENSSL    ${OPENSSL_FOUND})
if(OPENSSL_FOUND)
  set(OPENSSL_INCLUDE_DIRS  ${OPENSSL_INCLUDE_DIR})
else()
  set(OPENSSL_INCLUDE_DIRS  "")
  set(OPENSSL_LIBRARIES     "")
endif()
# libev (for src)
set(HAVE_LIBEV      ${LIBEV_FOUND})
set(HAVE_ZLIB       ${ZLIB_FOUND})
set(HAVE_LIBEVENT_OPENSSL ${LIBEVENT_FOUND})
if(LIBEVENT_FOUND)
  # Must both link the core and openssl libraries.
  set(LIBEVENT_OPENSSL_LIBRARIES ${LIBEVENT_LIBRARIES})
endif()
# jansson (for src/nghttp, src/deflatehd and src/inflatehd)
set(HAVE_JANSSON    ${JANSSON_FOUND})
# libxml2 (for src/nghttp)
set(HAVE_LIBXML2    ${LIBXML2_FOUND})
if(LIBXML2_FOUND)
  set(LIBXML2_INCLUDE_DIRS  ${LIBXML2_INCLUDE_DIR})
else()
  set(LIBXML2_INCLUDE_DIRS  "")
  set(LIBXML2_LIBRARIES     "")
endif()
# jemalloc
set(HAVE_JEMALLOC   ${JEMALLOC_FOUND})
# spdylay (for src/nghttpx and src/h2load)
set(HAVE_SPDYLAY    ${SPDYLAY_FOUND})

if(ENABLE_ASIO_LIB)
  find_package(Boost 1.54.0 REQUIRED system thread)
endif()

# The nghttp, nghttpd and nghttpx under src depend on zlib, OpenSSL and libev
if(ENABLE_APP AND NOT (ZLIB_FOUND AND OPENSSL_FOUND AND LIBEV_FOUND))
  message(FATAL_ERROR "Applications were requested (ENABLE_APP=1) but dependencies are not met.")
endif()

# HPACK tools requires jansson
if(ENABLE_HPACK_TOOLS AND NOT HAVE_JANSSON)
  message(FATAL_ERROR "HPACK tools were requested (ENABLE_HPACK_TOOLS=1) but dependencies are not met.")
endif()

# C++ library libnghttp2_asio
if(ENABLE_EXAMPLES AND NOT (OPENSSL_FOUND AND LIBEVENT_OPENSSL_FOUND))
  message(FATAL_ERROR "examples were requested (ENABLE_EXAMPLES=1) but dependencies are not met.")
endif()

# third-party http-parser only be built when needed
if(ENABLE_EXAMPLES OR ENABLE_APP OR ENABLE_HPACK_TOOLS OR ENABLE_ASIO_LIB)
  set(ENABLE_THIRD_PARTY 1)
  # mruby (for src/nghttpx)
  set(HAVE_MRUBY      ${WITH_MRUBY})
  set(HAVE_NEVERBLEED ${WITH_NEVERBLEED})
else()
  set(HAVE_MRUBY 0)
  set(HAVE_NEVERBLEED 0)
endif()

# Checks for header files.
include(CheckIncludeFile)
check_include_file("arpa/inet.h"    HAVE_ARPA_INET_H)
check_include_file("fcntl.h"        HAVE_FCNTL_H)
check_include_file("inttypes.h"     HAVE_INTTYPES_H)
check_include_file("limits.h"       HAVE_LIMITS_H)
check_include_file("netdb.h"        HAVE_NETDB_H)
check_include_file("netinet/in.h"   HAVE_NETINET_IN_H)
check_include_file("pwd.h"          HAVE_PWD_H)
check_include_file("sys/socket.h"   HAVE_SYS_SOCKET_H)
check_include_file("sys/time.h"     HAVE_SYS_TIME_H)
check_include_file("syslog.h"       HAVE_SYSLOG_H)
check_include_file("time.h"         HAVE_TIME_H)
check_include_file("unistd.h"       HAVE_UNISTD_H)

include(CheckTypeSize)
# Checks for typedefs, structures, and compiler characteristics.
# AC_TYPE_SIZE_T
check_type_size("ssize_t" SIZEOF_SSIZE_T)
if(SIZEOF_SSIZE_T STREQUAL "")
  # ssize_t is a signed type in POSIX storing at least -1.
  # Set it to "int" to match the behavior of AC_TYPE_SSIZE_T (autotools).
  set(ssize_t int)
endif()
# AC_TYPE_UINT8_T
# AC_TYPE_UINT16_T
# AC_TYPE_UINT32_T
# AC_TYPE_UINT64_T
# AC_TYPE_INT8_T
# AC_TYPE_INT16_T
# AC_TYPE_INT32_T
# AC_TYPE_INT64_T
# AC_TYPE_OFF_T
# AC_TYPE_PID_T
# AC_TYPE_UID_T
# XXX To support inline for crappy compilers, see https://cmake.org/Wiki/CMakeTestInline
# AC_C_INLINE
# XXX is AC_SYS_LARGEFILE still needed for modern systems?
# add_definitions(-D_FILE_OFFSET_BITS=64)

include(CheckStructHasMember)
check_struct_has_member("struct tm" tm_gmtoff time.h HAVE_STRUCT_TM_TM_GMTOFF)

# Check size of pointer to decide we need 8 bytes alignment adjustment.
check_type_size("int *"   SIZEOF_INT_P)
check_type_size("time_t"  SIZEOF_TIME_T)

# Checks for library functions.
include(CheckFunctionExists)
check_function_exists(_Exit     HAVE__EXIT)
check_function_exists(accept4   HAVE_ACCEPT4)

# timerfd_create was added in linux kernel 2.6.25
include(CheckSymbolExists)
# XXX does this correctly detect initgroups (un)availability on cygwin?
check_symbol_exists(initgroups grp.h HAVE_DECL_INITGROUPS)
if(NOT HAVE_DECL_INITGROUPS AND HAVE_UNISTD_H)
  # FreeBSD declares initgroups() in unistd.h
  check_symbol_exists(initgroups unistd.h HAVE_DECL_INITGROUPS2)
  if(HAVE_DECL_INITGROUPS2)
    set(HAVE_DECL_INITGROUPS 1)
  endif()
endif()

check_function_exists(timerfd_create HAVE_TIMERFD_CREATE)
# Checks for epoll availability, primarily for examples/tiny-nghttpd
check_symbol_exists(epoll_create sys/epoll.h HAVE_EPOLL)
if(HAVE_EPOLL AND HAVE_TIMERFD_CREATE)
  set(ENABLE_TINY_NGHTTPD 1)
endif()

set(WARNCFLAGS)
set(WARNCXXFLAGS)
if(CMAKE_C_COMPILER_ID MATCHES "MSVC")
  if(ENABLE_WERROR)
    set(WARNCFLAGS    /WX)
    set(WARNCXXFLAGS  /WX)
  endif()
else()
  if(ENABLE_WERROR)
    extract_valid_c_flags(WARNCFLAGS    -Werror)
    extract_valid_c_flags(WARNCXXFLAGS  -Werror)
  endif()

  # For C compiler
  extract_valid_c_flags(WARNCFLAGS
    -Wall
    -Wextra
    -Wmissing-prototypes
    -Wstrict-prototypes
    -Wmissing-declarations
    -Wpointer-arith
    -Wdeclaration-after-statement
    -Wformat-security
    -Wwrite-strings
    -Wshadow
    -Winline
    -Wnested-externs
    -Wfloat-equal
    -Wundef
    -Wendif-labels
    -Wempty-body
    -Wcast-align
    -Wclobbered
    -Wvla
    -Wpragmas
    -Wunreachable-code
    -Waddress
    -Wattributes
    -Wdiv-by-zero
    -Wshorten-64-to-32

    -Wconversion
    -Wextended-offsetof
    -Wformat-nonliteral
    -Wlanguage-extension-token
    -Wmissing-field-initializers
    -Wmissing-noreturn
    -Wmissing-variable-declarations
    # Not used because we cannot change public structs
    # -Wpadded
    -Wsign-conversion
    # Not used because this basically disallows default case
    # -Wswitch-enum
    -Wunreachable-code-break
    -Wunused-macros
    -Wunused-parameter
    -Wredundant-decls
    # Only work with Clang for the moment
    -Wheader-guard
    # This is required because we pass format string as "const char*.
    -Wno-format-nonliteral
  )

  extract_valid_cxx_flags(WARNCXXFLAGS
    # For C++ compiler
    -Wall
    -Wformat-security
  )
endif()

if(ENABLE_DEBUG)
  set(DEBUGBUILD 1)
endif()

# Some platform does not have working std::future.  We disable
# threading for those platforms.
if(NOT ENABLE_THREADS OR NOT HAVE_STD_FUTURE)
  set(NOTHREADS 1)
endif()

add_definitions(-DHAVE_CONFIG_H)
configure_file(cmakeconfig.h.in config.h)
# autotools-compatible names
# Sphinx expects relative paths in the .rst files. Use the fact that the files
# below are all one directory level deep.
file(RELATIVE_PATH top_srcdir   "${CMAKE_BINARY_DIR}/dir" "${CMAKE_SOURCE_DIR}")
file(RELATIVE_PATH top_builddir "${CMAKE_BINARY_DIR}/dir" "${CMAKE_BINARY_DIR}")
set(abs_top_srcdir  "${CMAKE_SOURCE_DIR}")
set(abs_top_builddir "${CMAKE_BINARY_DIR}")
# libnghttp2.pc (pkg-config file)
set(prefix          "${CMAKE_INSTALL_PREFIX}")
set(exec_prefix     "${CMAKE_INSTALL_PREFIX}")
set(libdir          "${CMAKE_INSTALL_FULL_LIBDIR}")
set(includedir      "${CMAKE_INSTALL_FULL_INCLUDEDIR}")
set(VERSION         "${PACKAGE_VERSION}")
# For init scripts and systemd service file (in contrib/)
set(bindir          "${CMAKE_INSTALL_FULL_BINDIR}")
set(sbindir         "${CMAKE_INSTALL_FULL_SBINDIR}")
foreach(name
   lib/libnghttp2.pc
   lib/includes/nghttp2/nghttp2ver.h
   src/libnghttp2_asio.pc
   python/setup.py
   integration-tests/config.go
   integration-tests/setenv
   doc/conf.py
   doc/index.rst
   doc/package_README.rst
   doc/tutorial-client.rst
   doc/tutorial-server.rst
   doc/tutorial-hpack.rst
   doc/nghttpx-howto.rst
   doc/h2load-howto.rst
   doc/libnghttp2_asio.rst
   doc/python-apiref.rst
   doc/building-android-binary.rst
   doc/nghttp2.h.rst
   doc/nghttp2ver.h.rst
   doc/asio_http2.h.rst
   doc/asio_http2_server.h.rst
   doc/asio_http2_client.h.rst
   doc/contribute.rst
)
  configure_file("${name}.in" "${name}" @ONLY)
endforeach()

include_directories(
  "${CMAKE_BINARY_DIR}" # for config.h
)
# For use in src/CMakeLists.txt
set(PKGDATADIR "${CMAKE_INSTALL_FULL_DATADIR}/${CMAKE_PROJECT_NAME}")

install(FILES README.rst DESTINATION "${CMAKE_INSTALL_DOCDIR}")

add_subdirectory(lib)
#add_subdirectory(lib/includes)
add_subdirectory(third-party)
add_subdirectory(src)
#add_subdirectory(src/includes)
add_subdirectory(examples)
add_subdirectory(python)
add_subdirectory(tests)
#add_subdirectory(tests/testdata)
add_subdirectory(integration-tests)
add_subdirectory(doc)
add_subdirectory(contrib)
add_subdirectory(script)


string(TOUPPER "${CMAKE_BUILD_TYPE}" _build_type)
message(STATUS "summary of build options:

    Package version: ${VERSION}
    Library version: ${LT_CURRENT}:${LT_REVISION}:${LT_AGE}
    Install prefix:  ${CMAKE_INSTALL_PREFIX}
    Target system:   ${CMAKE_SYSTEM_NAME}
    Compiler:
      Build type:     ${CMAKE_BUILD_TYPE}
      C compiler:     ${CMAKE_C_COMPILER}
      CFLAGS:         ${CMAKE_C_FLAGS_${_build_type}} ${CMAKE_C_FLAGS}
      C++ compiler:   ${CMAKE_CXX_COMPILER}
      CXXFLAGS:       ${CMAKE_CXX_FLAGS_${_build_type}} ${CMAKE_CXX_FLAGS}
      WARNCFLAGS:     ${WARNCFLAGS}
      CXX1XCXXFLAGS:  ${CXX1XCXXFLAGS}
    Python:
      Python:         ${PYTHON_EXECUTABLE}
      PYTHON_VERSION: ${PYTHON_VERSION_STRING}
      Library version:${PYTHONLIBS_VERSION_STRING}
      Cython:         ${CYTHON_EXECUTABLE}
    Test:
      CUnit:          ${HAVE_CUNIT} (LIBS='${CUNIT_LIBRARIES}')
      Failmalloc:     ${ENABLE_FAILMALLOC}
    Libs:
      OpenSSL:        ${HAVE_OPENSSL} (LIBS='${OPENSSL_LIBRARIES}')
      Libxml2:        ${HAVE_LIBXML2} (LIBS='${LIBXML2_LIBRARIES}')
      Libev:          ${HAVE_LIBEV} (LIBS='${LIBEV_LIBRARIES}')
      Libevent(SSL):  ${HAVE_LIBEVENT_OPENSSL} (LIBS='${LIBEVENT_OPENSSL_LIBRARIES}')
      Spdylay:        ${HAVE_SPDYLAY} (LIBS='${SPDYLAY_LIBRARIES}')
      Jansson:        ${HAVE_JANSSON} (LIBS='${JANSSON_LIBRARIES}')
      Jemalloc:       ${HAVE_JEMALLOC} (LIBS='${JEMALLOC_LIBRARIES}')
      Zlib:           ${HAVE_ZLIB} (LIBS='${ZLIB_LIBRARIES}')
      Boost::System:  ${Boost_SYSTEM_LIBRARY}
      Boost::Thread:  ${Boost_THREAD_LIBRARY}
    Third-party:
      http-parser:    ${ENABLE_THIRD_PARTY}
      MRuby:          ${HAVE_MRUBY}
      Neverbleed:     ${HAVE_NEVERBLEED}
    Features:
      Applications:   ${ENABLE_APP}
      HPACK tools:    ${ENABLE_HPACK_TOOLS}
      Libnghttp2_asio:${ENABLE_ASIO_LIB}
      Examples:       ${ENABLE_EXAMPLES}
      Python bindings:${ENABLE_PYTHON_BINDINGS}
      Threading:      ${ENABLE_THREADS}
")
if(ENABLE_LIB_ONLY_DISABLED_OTHERS)
  message("Only the library will be built. To build other components "
    "(such as applications and examples), set ENABLE_LIB_ONLY=OFF.")
endif()