cmake_minimum_required(VERSION 3.5)

# use, i.e. don't skip the full RPATH for the build tree
set(CMAKE_SKIP_BUILD_RPATH  FALSE)

# when building, don't use the install RPATH already
# (but later on when installing)
set(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE)

set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
set(CMAKE_MACOSX_RPATH 1)

# add the automatically determined parts of the RPATH
# which point to directories outside the build tree to the install RPATH
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

project(syslog-ng C)

set(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake/Modules ${PROJECT_SOURCE_DIR}/cmake/)
set(CMAKE_POSITION_INDEPENDENT_CODE 1)
set(BISON_FLAGS "-Wno-other")

include(CheckIncludeFiles)
include(ExternalProject)
include(external_or_find_package)
include(add_tests)

set(MEMORYCHECK_SUPPRESSIONS_FILE "${PROJECT_SOURCE_DIR}/tests/valgrind/unit-test-leak.supp")
set(MEMORYCHECK_COMMAND_OPTIONS "--num-callers=30 --sim-hints=no-nptl-pthread-stackcache --gen-suppressions=all --leak-check=full --freelist-vol=200000000 --freelist-big-blocks=10000000 --malloc-fill=55 --free-fill=AA")

file(STRINGS "${PROJECT_SOURCE_DIR}/VERSION" VERSION_FROM_FILE)
set(SYSLOG_NG_VERSION "${VERSION_FROM_FILE}")
set(SYSLOG_NG_COMBINED_VERSION ${SYSLOG_NG_VERSION})
set(SYSLOG_NG_SOURCE_REVISION ${SYSLOG_NG_VERSION})

set(SYSLOG_NG_PATH_PREFIX ${CMAKE_INSTALL_PREFIX})
set(SYSLOG_NG_PATH_SYSCONFDIR "\${prefix}/etc")
set(SYSLOG_NG_PATH_DATADIR "\${datarootdir}")
set(SYSLOG_NG_PATH_PIDFILEDIR "\${localstatedir}")
set(SYSLOG_NG_PATH_LOCALSTATEDIR "\${prefix}/var")
set(SYSLOG_NG_MODULE_PATH "\${exec_prefix}/lib/syslog-ng")
set(SYSLOG_NG_PATH_EXECPREFIX "\${prefix}")
set(SYSLOG_NG_PATH_CONFIG_INCLUDEDIR "\${datadir}/syslog-ng/include")
set(SYSLOG_NG_PATH_SCLDIR "\${datadir}/syslog-ng/include/scl")
set(SYSLOG_NG_PATH_LIBEXECDIR "\${exec_prefix}/libexec")
set(SYSLOG_NG_PATH_DATAROOTDIR "\${prefix}/share")
set(SYSLOG_NG_ENABLE_LINUX_CAPS 0)
set(SYSLOG_NG_ENABLE_TCP_WRAPPER 0)
set(SYSLOG_NG_ENABLE_GPROF 0)
set(SYSLOG_NG_ENABLE_MEMTRACE 0)
set(SYSLOG_NG_ENABLE_SYSTEMD 0)
set(SYSLOG_NG_PATH_MODULEDIR "\${exec_prefix}/lib/syslog-ng")
set(SYSLOG_NG_PACKAGE_NAME "${CMAKE_PROJECT_NAME}")
set(SYSLOG_NG_PATH_XSDDIR "\${datadir}/syslog-ng/xsd")
set(SYSLOG_NG_JAVA_MODULE_PATH "\${exec_prefix}/lib/syslog-ng/java-modules")
set(SYSLOG_NG_PATH_TOPSRC_DIR "${PROJECT_SOURCE_DIR}")
set(LIBDIR "\${exec_prefix}/lib")
set(INCLUDEDIR "\${prefix}/include")
set(TOOLSDIR "\${datadir}/syslog-ng/tools")
set(SYSLOG_NG_ENABLE_FORCED_SERVER_MODE 1)

set (LOGGEN_PLUGIN_INSTALL_DIR "${CMAKE_INSTALL_PREFIX}/lib/syslog-ng/loggen")

IF(CMAKE_BUILD_TYPE MATCHES Debug OR CMAKE_BUILD_TYPE MATCHES RelWithDebInfo)
  set(SYSLOG_NG_ENABLE_DEBUG 1)
ELSE()
  set(SYSLOG_NG_ENABLE_DEBUG 0)
ENDIF()

include(CheckTypeSize)
include(CheckStructMember)
include(CheckSymbolExists)
include(GenerateYFromYm)
include(CheckStructHasMember)

add_definitions(-D_GNU_SOURCE=1)

include(CheckSockaddrStorage)
if (HAVE_STRUCT_SOCKADDR_STORAGE)
    set(SYSLOG_NG_HAVE_STRUCT_SOCKADDR_STORAGE 1)
else()
    set(SYSLOG_NG_HAVE_STRUCT_SOCKADDR_STORAGE 0)
endif()

include(CheckCreds)
set(SYSLOG_NG_HAVE_STRUCT_UCRED (HAVE_STRUCT_UCRED OR HAVE_STRUCT_CMSGCRED))
check_struct_member ("struct msghdr" "msg_control" "sys/types.h;sys/socket.h" SYSLOG_NG_HAVE_CTRLBUF_IN_MSGHDR)

include(CheckIPv6)

option(ENABLE_IPV6 "Enable IPv6" ON)
if (ENABLE_IPV6)
    set(SYSLOG_NG_ENABLE_IPV6 ${HAVE_IPV6})
endif()

check_symbol_exists(strtoll stdlib.h SYSLOG_NG_HAVE_STRTOLL)
check_symbol_exists(strnlen string.h SYSLOG_NG_HAVE_STRNLEN)
check_symbol_exists(strtoimax inttypes.h SYSLOG_NG_HAVE_STRTOIMAX)
check_symbol_exists(inet_aton "sys/socket.h;netinet/in.h;arpa/inet.h" SYSLOG_NG_HAVE_INET_ATON)
check_symbol_exists(getutent utmp.h SYSLOG_NG_HAVE_GETUTENT)
check_symbol_exists(getutxent utmpx.h SYSLOG_NG_HAVE_GETUTXENT)
check_symbol_exists(getaddrinfo "netdb.h;sys/socket.h;sys/types.h" SYSLOG_NG_HAVE_GETADDRINFO)
check_symbol_exists(getnameinfo "netdb.h;sys/socket.h" SYSLOG_NG_HAVE_GETNAMEINFO)
check_symbol_exists(clock_gettime "time.h" SYSLOG_NG_HAVE_CLOCK_GETTIME)

check_include_files(utmp.h SYSLOG_NG_HAVE_UTMP_H)
check_include_files(utmpx.h SYSLOG_NG_HAVE_UTMPX_H)

check_struct_has_member("struct utmpx" "ut_type" "utmpx.h" UTMPX_HAS_UT_TYPE LANGUAGE C)
check_struct_has_member("struct utmp" "ut_type" "utmp.h" UTMP_HAS_UT_TYPE LANGUAGE C)
check_struct_has_member("struct utmpx" "ut_user" "utmpx.h" UTMPX_HAS_UT_USER LANGUAGE C)
check_struct_has_member("struct utmp" "ut_user" "utmp.h" UTMP_HAS_UT_USER LANGUAGE C)

if ((UTMPX_HAS_UT_TYPE AND UTMPX_HAS_UT_USER) OR (UTMPX_HAS_UT_TYPE AND UTMP_HAS_UT_USER))
  set(SYSLOG_NG_HAVE_MODERN_UTMP 1)
endif()

set(WITH_GETTEXT "" CACHE STRING "Set the prefix where gettext is installed (e.g. /usr)")

set(CMAKE_C_STANDARD 99)

find_package(BISON 2.4 REQUIRED)
find_package(GLIB COMPONENTS gmodule gthread 2.10.1 REQUIRED)

set (SYSLOG_NG_USE_CONST_IVYKIS_MOCK 1)
external_or_find_package(IVYKIS REQUIRED)
if ((NOT IVYKIS_INTERNAL) AND (IVYKIS_PKGCONF_VERSION VERSION_LESS "0.39"))
  set (SYSLOG_NG_USE_CONST_IVYKIS_MOCK 0)
endif()

find_package(OpenSSL REQUIRED)
find_package(FLEX REQUIRED)
find_package(LibNet REQUIRED)
find_package(Resolv REQUIRED)
find_package(Wrap)
find_package(criterion)
find_package(Inotify)
set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)
set(SYSLOG_NG_HAVE_INOTIFY "${Inotify_FOUND}")

set (PYTHON_VERSION "AUTO" CACHE STRING "Version of the installed development library" )

if ("${PYTHON_VERSION}" STREQUAL "AUTO")
  find_package(PythonInterp)
  find_package(PythonLibs)
else ()
  find_package(PythonInterp EXACT "${PYTHON_VERSION}" REQUIRED)
  find_package(PythonLibs EXACT "${PYTHON_VERSION}" REQUIRED)
endif ()

include(openssl_functions)
openssl_set_defines()

pkg_check_modules(LIBPCRE REQUIRED libpcre)

if (WRAP_FOUND)
  set(SYSLOG_NG_ENABLE_TCP_WRAPPER 1)
endif()

if (LIBNET_FOUND)
  set(SYSLOG_NG_ENABLE_SPOOF_SOURCE 1)
endif()

if (WITH_GETTEXT)
    set(CMAKE_PREFIX_PATH ${WITH_GETTEXT})
    find_package(Gettext REQUIRED QUIET)
    set(CMAKE_PREFIX_PATH "")
else()
    find_package(Gettext REQUIRED QUIET)
endif()

set(Eventlog_INCLUDE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/lib/eventlog/src")

add_custom_target(style-check  COMMAND ${PROJECT_SOURCE_DIR}/scripts/style-checker.sh check  ${PROJECT_SOURCE_DIR} ${PROJECT_BINARY_DIR})
add_custom_target(style-format COMMAND ${PROJECT_SOURCE_DIR}/scripts/style-checker.sh format ${PROJECT_SOURCE_DIR} ${PROJECT_BINARY_DIR})
add_custom_target(check-copyright
  COMMAND ${PROJECT_SOURCE_DIR}/tests/copyright/check.sh . ${PROJECT_BINARY_DIR} policy
  WORKING_DIRECTORY ${PROJECT_SOURCE_DIR})
set_target_properties(check-copyright PROPERTIES
  ADDITIONAL_MAKE_CLEAN_FILES
  "copyright-run.log;copyright-err.log")

# The inclusion of CTest triggers enable_testing()
# CMake will generate tests only if the enable_testing() command has been invoked.
# The CTest module invokes the command automatically when the BUILD_TESTING option is ON.

if (CRITERION_FOUND)
  option(BUILD_TESTING "Enable unit tests" ON)
else()
  option(BUILD_TESTING  "Enable unit tests" OFF)
endif()

if (BUILD_TESTING AND NOT CRITERION_FOUND)
  message(FATAL_ERROR "BUILD_TESTING enabled without criterion detected!")
endif()

if (CRITERION_FOUND)
  set(CTEST_ENVIRONMENT
    "G_SLICE=always-malloc,debug-blocks"
    "G_DEBUG=fatal-warnings,fatal-criticals,gc-friendly")
  include(CTest)
endif()

set(IMPORTANT_WARNINGS
  -Wshadow
  -fcommon)

set(ACCEPTABLE_WARNINGS
  -Wno-stack-protector
  -Wno-unused-parameter
  -Wno-variadic-macros)

option(ENABLE_EXTRA_WARNINGS "Enable extra warnings" ON)
if (ENABLE_EXTRA_WARNINGS)
  set(EXTRA_WARNINGS
    -Wimplicit-function-declaration
    -Wnested-externs
    -Wold-style-declaration
    -Wstrict-prototypes
    -Wswitch-default
    -Wall
    -Wmissing-parameter-type
    -Wuninitialized
    -Wunused-but-set-parameter
    -Wcast-align
    -Wdeprecated
    -Wdeprecated-declarations
    -Woverflow
    -Wdouble-promotion
    -Wfloat-equal)
endif()

add_compile_options(${IMPORTANT_WARNINGS} ${ACCEPTABLE_WARNINGS} ${EXTRA_WARNINGS})

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/syslog-ng-config.h.in ${CMAKE_CURRENT_BINARY_DIR}/syslog-ng-config.h)
include_directories(${CMAKE_CURRENT_BINARY_DIR})

add_subdirectory(Mk)
add_subdirectory(scl)
add_subdirectory(lib)
add_subdirectory(modules)
add_subdirectory(scripts)
add_subdirectory(syslog-ng)
add_subdirectory(syslog-ng-ctl)
add_test_subdirectory(libtest)
add_subdirectory(tests)

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/syslog-ng-config.h DESTINATION include/syslog-ng)

install(DIRECTORY DESTINATION var)

# Package Information for pkg-config
set(PKG_CONFIG_EXEC_PREFIX ${SYSLOG_NG_PATH_EXECPREFIX})
set(PKG_CONFIG_DATAROOTDIR ${SYSLOG_NG_PATH_DATAROOTDIR})
set(PKG_CONFIG_DATADIR ${SYSLOG_NG_PATH_DATAROOTDIR})
set(PKG_CONFIG_LIBDIR ${LIBDIR})
set(PKG_CONFIG_INCLUDEDIR ${INCLUDEDIR})
set(PKG_CONFIG_TOOLSDIR ${TOOLSDIR})
set(PKG_CONFIG_MODULEDIR ${SYSLOG_NG_PATH_MODULEDIR})
set(PKG_CONFIG_CONFIG_INCLUDEDIR ${SYSLOG_NG_PATH_CONFIG_INCLUDEDIR})
set(PKG_CONFIG_SCLDIR ${SYSLOG_NG_PATH_SCLDIR})
set(PKG_CONFIG_IVYKIS ${IVYKIS_INCLUDE_DIRS})
set(PKG_CONFIG_PACKAGE_VERSION ${SYSLOG_NG_VERSION})
set(PKG_CONFIG_INTERNAL_IVYKIS_CFLAGS ${IVYKIS_INCLUDE_DIRS})
set(libdir "\${libdir}")
set(includedir "\${includedir}")

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/syslog-ng.pc.cmake ${CMAKE_CURRENT_BINARY_DIR}/syslog-ng.pc)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/syslog-ng.pc DESTINATION lib/pkgconfig)
