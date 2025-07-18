cmake_minimum_required(VERSION 3.11)
project(hiredis-cluster)
include(GNUInstallDirs)

# Options
option(DOWNLOAD_HIREDIS  "Download the dependency hiredis from GitHub" ON)
option(ENABLE_SSL        "Enable SSL/TLS support" OFF)
option(DISABLE_TESTS     "Disable compilation of test" OFF)
option(ENABLE_IPV6_TESTS "Enable IPv6 tests requiring special prerequisites" OFF)
option(ENABLE_COVERAGE   "Enable test coverage reporting" OFF)

macro(getVersionBit name)
  set(VERSION_REGEX "^#define ${name} (.+)$")
  file(STRINGS "${CMAKE_CURRENT_SOURCE_DIR}/hircluster.h"
    VERSION_BIT REGEX ${VERSION_REGEX})
  string(REGEX REPLACE ${VERSION_REGEX} "\\1" ${name} "${VERSION_BIT}")
endmacro(getVersionBit)

# Get version information from src
getVersionBit(HIREDIS_CLUSTER_MAJOR)
getVersionBit(HIREDIS_CLUSTER_MINOR)
getVersionBit(HIREDIS_CLUSTER_PATCH)
getVersionBit(HIREDIS_CLUSTER_SONAME)
set(VERSION "${HIREDIS_CLUSTER_MAJOR}.${HIREDIS_CLUSTER_MINOR}.${HIREDIS_CLUSTER_PATCH}")
message("Detected version: ${VERSION}")

project(hiredis-cluster
  VERSION "${VERSION}"
  LANGUAGES C)

# Use plain C99 (-std=c99) without extensions like GNU C (-std=gnu99)
set(CMAKE_C_STANDARD 99)
set(CMAKE_C_EXTENSIONS OFF)

# Build using a sanitizer
if(USE_SANITIZER)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fno-omit-frame-pointer -fsanitize=${USE_SANITIZER}")
endif()

if(ENABLE_COVERAGE)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} --coverage -O0" )
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} --coverage" )
endif()

SET(hiredis_cluster_sources
    adlist.c
    command.c
    crc16.c
    dict.c
    hiarray.c
    hircluster.c
    hiutil.c)

if(WIN32 OR MINGW)
    add_compile_definitions(_CRT_SECURE_NO_WARNINGS WIN32_LEAN_AND_MEAN)
endif()

add_library(hiredis_cluster
  SHARED
  ${hiredis_cluster_sources})

if(NOT MSVC)
  target_compile_options(hiredis_cluster PRIVATE -Wall -Wextra -pedantic -Werror
    -Wstrict-prototypes -Wwrite-strings -Wno-missing-field-initializers)

  # Add extra defines when CMAKE_BUILD_TYPE is set to Debug
  set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -DHI_ASSERT_PANIC -DHI_HAVE_BACKTRACE")
  # Alternative: -DHI_ASSERT_LOG)
endif()

set_target_properties(hiredis_cluster
    PROPERTIES
    WINDOWS_EXPORT_ALL_SYMBOLS TRUE
    VERSION "${HIREDIS_CLUSTER_SONAME}")

if(DOWNLOAD_HIREDIS)
  if(${CMAKE_VERSION} VERSION_LESS "3.20")
    message(FATAL_ERROR
      "Downloading of the dependency 'hiredis' requires CMake >= v3.20.\n"
      "Upgrade CMake or manually install 'hiredis' and use -DDOWNLOAD_HIREDIS=OFF")
  endif()

  set(HIREDIS_VERSION "1.3.0")
  message("Downloading dependency: hiredis v${HIREDIS_VERSION}")

  include(FetchContent)
  FetchContent_Declare(hiredis
    GIT_REPOSITORY https://github.com/redis/hiredis
    GIT_TAG        "v${HIREDIS_VERSION}"
    SOURCE_DIR     "${CMAKE_CURRENT_BINARY_DIR}/_deps/hiredis"
  )

  # Disable tests in hiredis
  set(DISABLE_TESTS_OLD ${DISABLE_TESTS})
  set(DISABLE_TESTS ON CACHE INTERNAL "")
  FetchContent_GetProperties(hiredis)
  if(NOT hiredis_POPULATED)
    FetchContent_Populate(hiredis)
    add_subdirectory(${hiredis_SOURCE_DIR} ${hiredis_BINARY_DIR})
  endif()
  set(DISABLE_TESTS ${DISABLE_TESTS_OLD} CACHE INTERNAL "")

  # Create an empty *-config.cmake for find_package
  # See: https://github.com/abandonware-pjz37/cmake-find-package-include/blob/master/hooks/fetch.cmake
  set(stub_dir "${CMAKE_CURRENT_BINARY_DIR}/generated/pkg")

  file(WRITE "${stub_dir}/hiredis-config.cmake" "")
  file(WRITE "${stub_dir}/hiredis-config-version.cmake" "set(PACKAGE_VERSION ${HIREDIS_VERSION})")
  set(hiredis_DIR ${stub_dir})
  # Set variables normally got from hiredis-config.cmake
  set(hiredis_LIBRARIES hiredis::hiredis)
  set(hiredis_INCLUDE_DIRS "${CMAKE_CURRENT_BINARY_DIR}/_deps")

  if(ENABLE_SSL)
    file(WRITE "${stub_dir}/hiredis_ssl-config.cmake" "")
    set(hiredis_ssl_DIR ${stub_dir})
  endif()

else()
  message("Expecting to find dependencies in path..")
endif()

find_package(hiredis QUIET)
if(NOT hiredis_FOUND)
  message("CMake package for 'hiredis' not found, searching for the library..")
  find_library(HIREDIS_LIB hiredis REQUIRED)
  find_path(HIREDIS_INCLUDES hiredis/hiredis.h)
  add_library(hiredis::hiredis UNKNOWN IMPORTED GLOBAL)
  set_target_properties(hiredis::hiredis PROPERTIES IMPORTED_LOCATION ${HIREDIS_LIB})
  set_target_properties(hiredis::hiredis PROPERTIES INTERFACE_INCLUDE_DIRECTORIES ${HIREDIS_INCLUDES})
endif()

if(NOT TARGET hiredis::hiredis)
  # Add target to support older hiredis releases
  add_library(hiredis::hiredis ALIAS hiredis)
endif()

target_include_directories(hiredis_cluster PUBLIC
  $<BUILD_INTERFACE:${hiredis_INCLUDE_DIRS}>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
  $<INSTALL_INTERFACE:include>)

if(WIN32 OR MINGW)
  target_link_libraries(hiredis_cluster PUBLIC ws2_32 hiredis::hiredis)
else()
  target_link_libraries(hiredis_cluster PUBLIC hiredis::hiredis)
endif()

if(ENABLE_SSL)
  find_package(hiredis_ssl QUIET)
  if(NOT hiredis_ssl_FOUND)
    message("CMake package for 'hiredis_ssl' not found, searching for the library..")
    find_library(HIREDIS_SSL_LIB hiredis_ssl REQUIRED)
    find_path(HIREDIS_SSL_INCLUDES hiredis/hiredis_ssl.h)
    add_library(hiredis::hiredis_ssl UNKNOWN IMPORTED GLOBAL)
    set_target_properties(hiredis::hiredis_ssl PROPERTIES IMPORTED_LOCATION ${HIREDIS_SSL_LIB})
    set_target_properties(hiredis::hiredis_ssl PROPERTIES INTERFACE_INCLUDE_DIRECTORIES ${HIREDIS_SSL_INCLUDES})
  endif()

  if(NOT TARGET hiredis::hiredis_ssl)
    # Add target to support older hiredis releases
    add_library(hiredis::hiredis_ssl ALIAS hiredis_ssl)
  endif()

  add_library(hiredis_cluster_ssl
    SHARED hircluster_ssl.c)
  set_target_properties(hiredis_cluster_ssl
    PROPERTIES
    WINDOWS_EXPORT_ALL_SYMBOLS TRUE
    VERSION "${HIREDIS_CLUSTER_SONAME}")
  target_link_libraries(hiredis_cluster_ssl
    PRIVATE hiredis_cluster
    PUBLIC hiredis::hiredis_ssl)
endif()

if(NOT DISABLE_TESTS)
  include(CTest)
  add_subdirectory(tests)
endif()

# Code formatting target
find_program(CLANG_FORMAT "clang-format")
file(GLOB_RECURSE FILES_TO_FORMAT
  ${PROJECT_SOURCE_DIR}/*.[ch]
)
add_custom_target(format
  COMMAND ${CLANG_FORMAT} -i ${FILES_TO_FORMAT}
)

# Code coverage target
if(ENABLE_COVERAGE)
  find_program(GCOVR "gcovr")

  add_custom_command(OUTPUT _run_gcovr
    POST_BUILD
    COMMAND ${GCOVR} -r ${CMAKE_SOURCE_DIR} --object-dir=${CMAKE_BINARY_DIR} --html-details coverage.html
    COMMAND echo "Coverage report generated: ${CMAKE_BINARY_DIR}/coverage.html"
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR})
  add_custom_target (coverage DEPENDS _run_gcovr)
endif()

configure_file(hiredis_cluster.pc.in hiredis_cluster.pc @ONLY)

install(TARGETS hiredis_cluster
  EXPORT hiredis_cluster-targets
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})

install(FILES hircluster.h adlist.h hiarray.h dict.h
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/hiredis_cluster)

install(DIRECTORY adapters
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/hiredis_cluster)

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/hiredis_cluster.pc
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)

export(EXPORT hiredis_cluster-targets
  FILE ${CMAKE_CURRENT_BINARY_DIR}/hiredis_cluster-targets.cmake
  NAMESPACE hiredis_cluster::)

set(CMAKE_CONF_INSTALL_DIR share/hiredis_cluster)
set(INCLUDE_INSTALL_DIR include)
include(CMakePackageConfigHelpers)
configure_package_config_file(hiredis_cluster-config.cmake.in ${CMAKE_CURRENT_BINARY_DIR}/hiredis_cluster-config.cmake
  INSTALL_DESTINATION ${CMAKE_CONF_INSTALL_DIR}
  PATH_VARS INCLUDE_INSTALL_DIR)
write_basic_package_version_file(${CMAKE_CURRENT_BINARY_DIR}/hiredis_cluster-config-version.cmake
  COMPATIBILITY SameMinorVersion)

install(EXPORT hiredis_cluster-targets
  FILE hiredis_cluster-targets.cmake
  NAMESPACE hiredis_cluster::
  DESTINATION ${CMAKE_CONF_INSTALL_DIR})

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/hiredis_cluster-config.cmake
              ${CMAKE_CURRENT_BINARY_DIR}/hiredis_cluster-config-version.cmake
  DESTINATION ${CMAKE_CONF_INSTALL_DIR})

# Install target for hiredis_cluster_ssl
if(ENABLE_SSL)
  configure_file(hiredis_cluster_ssl.pc.in hiredis_cluster_ssl.pc @ONLY)

  install(TARGETS hiredis_cluster_ssl
    EXPORT hiredis_cluster_ssl-targets
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})
  install(FILES hircluster_ssl.h
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/hiredis_cluster)
  install(FILES ${CMAKE_CURRENT_BINARY_DIR}/hiredis_cluster_ssl.pc
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
  export(EXPORT hiredis_cluster_ssl-targets
    FILE ${CMAKE_CURRENT_BINARY_DIR}/hiredis_cluster_ssl-targets.cmake
    NAMESPACE hiredis_cluster::)
  set(CMAKE_CONF_INSTALL_DIR share/hiredis_cluster_ssl)
  configure_package_config_file(hiredis_cluster_ssl-config.cmake.in ${CMAKE_CURRENT_BINARY_DIR}/hiredis_cluster_ssl-config.cmake
    INSTALL_DESTINATION ${CMAKE_CONF_INSTALL_DIR}
    PATH_VARS INCLUDE_INSTALL_DIR)
  install(EXPORT hiredis_cluster_ssl-targets
    FILE hiredis_cluster_ssl-targets.cmake
    NAMESPACE hiredis_cluster::
    DESTINATION ${CMAKE_CONF_INSTALL_DIR})
  install(FILES ${CMAKE_CURRENT_BINARY_DIR}/hiredis_cluster_ssl-config.cmake
    DESTINATION ${CMAKE_CONF_INSTALL_DIR})
endif()
