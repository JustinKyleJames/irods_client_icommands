cmake_minimum_required(VERSION 3.14...3.18 FATAL_ERROR)
if (POLICY CMP0109)
  # find_program requires permission to execute but not to read
  cmake_policy(SET CMP0109 NEW)
endif()
if (POLICY CMP0121)
  # Detect invalid indices in list()
  cmake_policy(SET CMP0121 NEW)
endif()
if (POLICY CMP0125)
  # Consistent behavior for cache variables managed by find_*()
  cmake_policy(SET CMP0125 NEW)
endif()
if (POLICY CMP0130)
  # Diagnose condition evaluation errors in while()
  cmake_policy(SET CMP0130 NEW)
endif()
if (POLICY CMP0132)
  # Consistent handling of compiler environment variables
  cmake_policy(SET CMP0132 NEW)
endif()
if (POLICY CMP0140)
  # Check validity of return() params
  cmake_policy(SET CMP0140 NEW)
endif()

option(IRODS_ENABLE_ADDRESS_SANITIZER "Enables detection of memory leaks and other features provided by Address Sanitizer." OFF)
option(IRODS_ENABLE_UNDEFINED_BEHAVIOR_SANITIZER "Enables detection of undefined behavior provided by Undefined Behavior Sanitizer." OFF)
option(IRODS_ENABLE_UNDEFINED_BEHAVIOR_SANITIZER_IMPLICIT_CONVERSION_CHECK "Enables detection of implicit conversions by Undefined Behavior Sanitizer." OFF)

find_package(IRODS 5.0.1 EXACT REQUIRED)
set(IRODS_PACKAGE_PREFIX "${PACKAGE_PREFIX_DIR}")

set(IRODS_PACKAGE_REVISION "0")
set(IRODS_STAMP_PACKAGE_REVISION OFF)

include(RequireOutOfSourceBuild)
include(IrodsCXXCompiler)

set(CMAKE_CXX_STANDARD ${IRODS_CXX_STANDARD})
# export-dynamic so stacktrace entries from executables have function names
set(CMAKE_EXE_LINKER_FLAGS_INIT "-rdynamic -Wl,--export-dynamic -Wl,--enable-new-dtags -Wl,--as-needed")
set(CMAKE_EXE_LINKER_FLAGS_RELEASE_INIT "-Wl,--gc-sections -Wl,-z,combreloc")
include(IrodsRunpathDefaults)

if (IRODS_ENABLE_ADDRESS_SANITIZER)
  add_compile_options(-fsanitize=address)
  add_link_options(-fsanitize=address)
endif()

if (IRODS_ENABLE_UNDEFINED_BEHAVIOR_SANITIZER)
  add_compile_options(
    -fsanitize=undefined
    -fsanitize=float-divide-by-zero
    -fsanitize=unsigned-integer-overflow
    -fsanitize=local-bounds
    -fsanitize=nullability
    -fsanitize-ignorelist=${CMAKE_CURRENT_SOURCE_DIR}/ub-sanitizer-ignorelist.txt)
  add_link_options(
    -fsanitize=undefined
    -fsanitize=float-divide-by-zero
    -fsanitize=unsigned-integer-overflow
    -fsanitize=local-bounds
    -fsanitize=nullability)
  if (IRODS_ENABLE_UNDEFINED_BEHAVIOR_SANITIZER_IMPLICIT_CONVERSION_CHECK)
     add_compile_options(-fsanitize=implicit-conversion)
     add_link_options(-fsanitize=implicit-conversion)
  endif()
endif()

if (IRODS_ENABLE_ADDRESS_SANITIZER OR IRODS_ENABLE_UNDEFINED_BEHAVIOR_SANITIZER)
  add_compile_options(
    -fno-omit-frame-pointer
    -fno-optimize-sibling-calls
    -O1)
  add_link_options(
    -fno-omit-frame-pointer
    -fno-optimize-sibling-calls
    -O1)
else()
  set(CMAKE_EXE_LINKER_FLAGS_INIT "${CMAKE_EXE_LINKER_FLAGS_INIT} -Wl,-z,defs")
endif()

project(icommands
  VERSION "${IRODS_VERSION}"
  LANGUAGES CXX)

set(CMAKE_EXPORT_COMPILE_COMMANDS TRUE)

include(${IRODS_TARGETS_PATH})

include(GNUInstallDirs)

if (NOT CMAKE_CONFIGURATION_TYPES AND NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build {Debug, Release}." FORCE)
  message(STATUS "Setting unspecified CMAKE_BUILD_TYPE to '${CMAKE_BUILD_TYPE}'. This is the correct setting for normal builds.")
endif()

set(ICOMMANDS_INSTALL_DIRECTORY "${IRODS_CLIENTS_DIRECTORY}/icommands" CACHE PATH "Path to install icommands executables")
set(ICOMMANDS_INSTALL_SYMLINKS TRUE CACHE BOOL "Choose whether to install symlinks to icommands in CMAKE_INSTALL_BINDIR.")

if (CMAKE_CXX_COMPILER_ID MATCHES ".*Clang")
  set(IRODS_BUILD_WITH_WERROR_DEFAULT ON)
else()
  set(IRODS_BUILD_WITH_WERROR_DEFAULT OFF)
endif()

set(IRODS_BUILD_WITH_WERROR ${IRODS_BUILD_WITH_WERROR_DEFAULT} CACHE BOOL "Choose whether to compile with -Werror.")

find_package(nlohmann_json "3.6.1" REQUIRED)
find_package(fmt "8.1.1" REQUIRED)
find_package(Threads REQUIRED)
find_package(OpenSSL REQUIRED COMPONENTS Crypto SSL)

if (IRODS_BUILD_WITH_WERROR)
  add_compile_options(-Werror)
endif()

if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
  add_compile_options($<$<COMPILE_LANGUAGE:CXX>:-fpermissive>)
endif()

add_compile_options(-Wall -Wextra -Wno-unused-parameter)

if (IRODS_BUILD_WITH_WERROR AND CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
  add_compile_options(-Wno-error=unused-variable)
  add_compile_options(-Wno-error=unused-but-set-variable)
endif()

set(IRODS_BUILD_DOCS_DEFAULT ON)

# Find help2man
if (NOT DEFINED BUILD_DOCS OR BUILD_DOCS)
  if (NOT DEFINED HELP2MAN)
    set(HELP2MAN_SET_BY_CMAKE ON)
  endif()
  find_program(
    HELP2MAN
    NAMES help2man
    DOC "help2man location"
    )
  if (DEFINED HELP2MAN)
    message(STATUS "Found help2man: ${HELP2MAN}")
    if (HELP2MAN_SET_BY_CMAKE)
      # check version number, but only if find_program went fishing
      execute_process(
        COMMAND "${HELP2MAN}" --version
        OUTPUT_VARIABLE HELP2MAN_VERSION
        OUTPUT_STRIP_TRAILING_WHITESPACE
        )
      string(REGEX REPLACE "\n.*" "" HELP2MAN_VERSION "${HELP2MAN_VERSION}")
      string(REGEX REPLACE "[^ ]+ +[^ ]+ +" "" HELP2MAN_VERSION "${HELP2MAN_VERSION}")
      message(STATUS "Found help2man version ${HELP2MAN_VERSION}")
      if (HELP2MAN_VERSION VERSION_LESS "1.37")
        # don't cache unusable help2man
        unset(HELP2MAN CACHE)
        if (NOT DEFINED BUILD_DOCS)
          set(IRODS_BUILD_DOCS_DEFAULT OFF)
        else()
          message(SEND_ERROR "help2man version 1.37 or later required, cannot build docs")
        endif()
      endif()
    endif()
  else()
    if (NOT DEFINED BUILD_DOCS)
      set(IRODS_BUILD_DOCS_DEFAULT OFF)
    else()
      message(SEND_ERROR "help2man not found, cannot build docs")
    endif()
  endif()
endif()

# Find gzip
if (NOT DEFINED BUILD_DOCS OR BUILD_DOCS)
  if (NOT DEFINED GZIP)
    set(GZIP_SET_BY_CMAKE ON)
  endif()
  find_program(
    GZIP
    NAMES gzip pigz
    DOC "gzip location"
    )
  if (DEFINED GZIP)
    message(STATUS "Found gzip: ${GZIP}")
  else()
    if (NOT DEFINED BUILD_DOCS)
      set(IRODS_BUILD_DOCS_DEFAULT OFF)
    else()
      message(SEND_ERROR "gzip not found, cannot build docs")
    endif()
  endif()
endif()

set(BUILD_DOCS ${IRODS_BUILD_DOCS_DEFAULT} CACHE BOOL "Build icommand documentation")
if (BUILD_DOCS)
  file(MAKE_DIRECTORY "${CMAKE_BINARY_DIR}/man")
  add_custom_target(docs ALL)
endif()

add_custom_target(icommands)

set(
  IRODS_CLIENT_ICOMMANDS_EXECUTABLES
  iadmin
  iapitest
  ibun
  icd
  ichksum
  ichmod
  iclienthints
  icp
  ienv
  ierror
  iexit
  ifsck
  iget
  igroupadmin
  ihelp
  iinit
  ils
  ilsresc
  imcoll
  imeta
  imiscsvrinfo
  imkdir
  imv
  ipasswd
  iphymv
  ips
  iput
  ipwd
  iqdel
  iqmod
  iqstat
  iquery
  iquest
  iquota
  ireg
  irepl
  irm
  irmdir
  irmtrash
  irsync
  irule
  iscan
  istream
  isysmeta
  iticket
  itouch
  itree
  itrim
  iunreg
  iuserinfo
  izonereport
  )

foreach(EXECUTABLE IN LISTS IRODS_CLIENT_ICOMMANDS_EXECUTABLES)
  add_executable(
    ${EXECUTABLE}
    ${CMAKE_SOURCE_DIR}/src/${EXECUTABLE}.cpp
    )
  target_link_libraries(
    ${EXECUTABLE}
    PRIVATE
    irods_client
    irods_plugin_dependencies
    irods_common
    nlohmann_json::nlohmann_json
    fmt::fmt
    ${IRODS_EXTERNALS_FULLPATH_BOOST}/lib/libboost_filesystem.so
    ${IRODS_EXTERNALS_FULLPATH_BOOST}/lib/libboost_program_options.so
    ${IRODS_EXTERNALS_FULLPATH_BOOST}/lib/libboost_system.so
    ${CMAKE_DL_LIBS}
    )
  target_include_directories(
    ${EXECUTABLE}
    PRIVATE
    ${IRODS_INCLUDE_DIRS}
    ${IRODS_EXTERNALS_FULLPATH_BOOST}/include
    )
  target_compile_definitions(${EXECUTABLE} PRIVATE ${IRODS_COMPILE_DEFINITIONS} ${IRODS_COMPILE_DEFINITIONS_PRIVATE} BOOST_SYSTEM_NO_DEPRECATED)
  target_compile_options(${EXECUTABLE} PRIVATE -Wno-write-strings)
  if (CMAKE_VERSION VERSION_LESS "3.13.0")
    set_property(TARGET ${EXECUTABLE} APPEND PROPERTY LINK_FLAGS "-Wl,-z,origin")
  else()
    target_link_options(${EXECUTABLE} PRIVATE "LINKER:-z,origin")
  endif()
  add_dependencies(icommands ${EXECUTABLE})
  install(
    TARGETS
    ${EXECUTABLE}
    RUNTIME
    DESTINATION "${ICOMMANDS_INSTALL_DIRECTORY}"
    COMPONENT icommands
    )

  if (${EXECUTABLE} STREQUAL "iapitest")
    continue()
  endif()

  if (BUILD_DOCS)
    set(EXECUTABLE_MANPAGE "${CMAKE_BINARY_DIR}/man/${EXECUTABLE}.1")
    add_custom_command(
      OUTPUT ${EXECUTABLE_MANPAGE}
      DEPENDS ${EXECUTABLE}
      COMMENT "generating manpage ${EXECUTABLE}.1"
      COMMAND "${CMAKE_COMMAND}" -E env "LD_LIBRARY_PATH=$<TARGET_FILE_DIR:irods_client>:$ENV{LD_LIBRARY_PATH}" "${HELP2MAN}" -h -h -N -n "an iRODS iCommand" --version-string="iRODS-${IRODS_VERSION}" $<TARGET_FILE:${EXECUTABLE}> -o "${EXECUTABLE_MANPAGE}"
      )
    add_custom_command(
      OUTPUT ${EXECUTABLE_MANPAGE}.gz
      DEPENDS ${EXECUTABLE_MANPAGE}
      COMMENT "gzipping manpage ${EXECUTABLE}.1"
      COMMAND "${GZIP}" -f < "${EXECUTABLE_MANPAGE}" > "${EXECUTABLE_MANPAGE}.gz"
      )
    add_custom_target(${EXECUTABLE}-manpage DEPENDS ${EXECUTABLE_MANPAGE}.gz)
    add_dependencies(docs ${EXECUTABLE}-manpage)
    install(
      FILES
      ${EXECUTABLE_MANPAGE}.gz
      DESTINATION ${CMAKE_INSTALL_MANDIR}/man1
      COMPONENT manpages
      )
  endif()
endforeach()

if (ICOMMANDS_INSTALL_SYMLINKS)
  # Normalize both
  cmake_path(
    NORMAL_PATH ICOMMANDS_INSTALL_DIRECTORY
    OUTPUT_VARIABLE ICOMMANDS_INSTALL_DIRECTORY_NORM
  )
  cmake_path(
    NORMAL_PATH CMAKE_INSTALL_BINDIR
    OUTPUT_VARIABLE CMAKE_INSTALL_BINDIR_NORM
  )

  # Remove trailing slashes
  string(REGEX MATCH "^(.+)/+$" _regex_out "${ICOMMANDS_INSTALL_DIRECTORY_NORM}")
  if (NOT ("${CMAKE_MATCH_1}" STREQUAL ""))
    set(ICOMMANDS_INSTALL_DIRECTORY_NORM "${CMAKE_MATCH_1}")
  endif()
  string(REGEX MATCH "^(.+)/+$" _regex_out "${CMAKE_INSTALL_BINDIR_NORM}")
  if (NOT ("${CMAKE_MATCH_1}" STREQUAL ""))
    set(CMAKE_INSTALL_BINDIR_NORM "${CMAKE_MATCH_1}")
  endif()

  if (NOT (ICOMMANDS_INSTALL_DIRECTORY_NORM STREQUAL CMAKE_INSTALL_BINDIR_NORM))
    cmake_path(IS_ABSOLUTE ICOMMANDS_INSTALL_DIRECTORY_NORM ICOMMANDS_INSTALL_DIRECTORY_IS_ABSOLUTE)
    cmake_path(IS_ABSOLUTE CMAKE_INSTALL_BINDIR_NORM CMAKE_INSTALL_BINDIR_IS_ABSOLUTE)

    # Make relative ICOMMANDS_INSTALL_DIRECTORY
    if (ICOMMANDS_INSTALL_DIRECTORY_IS_ABSOLUTE AND NOT CMAKE_INSTALL_BINDIR_IS_ABSOLUTE)
      set(ICOMMANDS_LINK_PATH "${ICOMMANDS_INSTALL_DIRECTORY_NORM}")
    else()
      cmake_path(
        RELATIVE_PATH ICOMMANDS_INSTALL_DIRECTORY_NORM
        BASE_DIRECTORY "${CMAKE_INSTALL_BINDIR_NORM}"
        OUTPUT_VARIABLE ICOMMANDS_LINK_PATH
      )
    endif()

    if (NOT ("${ICOMMANDS_LINK_PATH}" STREQUAL ""))
      # add trailing slash
      if (NOT (ICOMMANDS_LINK_PATH STREQUAL "/"))
        set(ICOMMANDS_LINK_PATH "${ICOMMANDS_LINK_PATH}/")
      endif()

      add_custom_target(links ALL)

      set(ICOMMANDS_LINK_STAGING_DIR "${CMAKE_BINARY_DIR}/links-staging")
      add_custom_command(
        OUTPUT "${ICOMMANDS_LINK_STAGING_DIR}"
        COMMAND "${CMAKE_COMMAND}" -E "make_directory" "${ICOMMANDS_LINK_STAGING_DIR}"
      )

      foreach(EXECUTABLE IN LISTS IRODS_CLIENT_ICOMMANDS_EXECUTABLES)
        # Despite what CMake's documentation might say, the generator expression TARGET_FILE_NAME
        # cannot be used in the OUTPUT target for add_custom_command.
        # See https://gitlab.kitware.com/cmake/cmake/-/issues/23837
        set(EXECUTABLE_LINK_TEMPNAME "${EXECUTABLE}.link")
        set(EXECUTABLE_LINK_NAME "$<TARGET_FILE_NAME:${EXECUTABLE}>")

        set(EXECUTABLE_LINK_STAGING "${ICOMMANDS_LINK_STAGING_DIR}/${EXECUTABLE_LINK_TEMPNAME}")
        set(EXECUTABLE_LINK_DEST_PATH "${ICOMMANDS_LINK_PATH}$<TARGET_FILE_NAME:${EXECUTABLE}>")

        add_custom_command(
          OUTPUT "${EXECUTABLE_LINK_STAGING}"
          DEPENDS "${ICOMMANDS_LINK_STAGING_DIR}"
          COMMENT "making symlink for ${EXECUTABLE}"
          COMMAND "${CMAKE_COMMAND}" -E create_symlink "${EXECUTABLE_LINK_DEST_PATH}" "${EXECUTABLE_LINK_STAGING}"
        )
        add_custom_target("${EXECUTABLE}-link" DEPENDS "${EXECUTABLE_LINK_STAGING}")
        add_dependencies(links "${EXECUTABLE}-link")
        install(
          PROGRAMS "${EXECUTABLE_LINK_STAGING}"
          DESTINATION "${CMAKE_INSTALL_BINDIR}"
          RENAME "${EXECUTABLE_LINK_NAME}"
          COMPONENT links
        )
      endforeach()
    endif()
  endif()
endif()

install(
  DIRECTORY ${CMAKE_SOURCE_DIR}/test
  DESTINATION ${IRODS_HOME_DIRECTORY}/clients/icommands
  COMPONENT testdata
  )


if(NOT ICOMMANDS_USERSPACE_TARBALL_PLATFORM_NAME)
  if(IRODS_LINUX_DISTRIBUTION_VERSION_CODENAME AND NOT IRODS_LINUX_DISTRIBUTION_VERSION_CODENAME STREQUAL "none")
    set(ICOMMANDS_USERSPACE_TARBALL_PLATFORM_NAME "${IRODS_LINUX_DISTRIBUTION_VERSION_CODENAME}")
  elseif(IRODS_LINUX_DISTRIBUTION_NAME)
    if(IRODS_LINUX_DISTRIBUTION_VERSION AND NOT IRODS_LINUX_DISTRIBUTION_VERSION STREQUAL "none")
      set(ICOMMANDS_USERSPACE_TARBALL_PLATFORM_NAME "${IRODS_LINUX_DISTRIBUTION_NAME}${IRODS_LINUX_DISTRIBUTION_VERSION}")
    else()
      set(ICOMMANDS_USERSPACE_TARBALL_PLATFORM_NAME "${IRODS_LINUX_DISTRIBUTION_NAME}")
    endif()
  else()
    set(ICOMMANDS_USERSPACE_TARBALL_PLATFORM_NAME "${CMAKE_SYSTEM_NAME}")
  endif()
endif()

set(ICOMMANDS_USERSPACE_TARBALL_FILENAME
    "irods-icommands-${icommands_VERSION}-${ICOMMANDS_USERSPACE_TARBALL_PLATFORM_NAME}-${CMAKE_SYSTEM_PROCESSOR}-${CMAKE_BUILD_TYPE}.tar.gz"
    CACHE STRING
    "Filename of userspace tarball package to generate")
set(ICOMMANDS_USERSPACE_TARBALL_ROOT_DIRNAME
    "irods-icommands-${icommands_VERSION}"
    CACHE STRING
    "Name of userspace tarball package's root directory. Set to empty string to use the root of the tarball as the root directory.")

set(ICOMMANDS_USERSPACE_TARBALL_PATH "${CMAKE_CURRENT_BINARY_DIR}/${ICOMMANDS_USERSPACE_TARBALL_FILENAME}")

# fetch the parts of irods-runtime currently exposed
set(irods_runtime_components "irods_client;irods_common;irods_plugin_dependencies")
foreach(irods_runtime_component ${irods_runtime_components})
  unset(iimported_location)
  get_target_property(iimported_location ${irods_runtime_component} LOCATION)
  list(APPEND IRODS_RUNTIME_LIBRARY_PATHS "${iimported_location}")
  unset(iimported_location)
endforeach()

file(
  GLOB ICOMMANDS_USERSPACE_TARBALL_INSTRUMENTATION
  LIST_DIRECTORIES false
  "${CMAKE_CURRENT_SOURCE_DIR}/packaging/userspace/*"
  )

add_custom_command(
  COMMAND "${CMAKE_CURRENT_SOURCE_DIR}/packaging/userspace/userspace-packager.py" -v
    "--cmake-path=${CMAKE_COMMAND}"
    "$<$<BOOL:${CMAKE_OBJDUMP}>:--objdump-path=${CMAKE_OBJDUMP}>"
    "$<$<BOOL:${CMAKE_READELF}>:--readelf-path=${CMAKE_READELF}>"
    "$<$<BOOL:${CMAKE_STRIP}>:--strip-path=${CMAKE_STRIP}>"
    "$<$<BOOL:${IRODS_LINUX_DISTRIBUTION_NAME}>:--target-platform=${IRODS_LINUX_DISTRIBUTION_NAME}>"
    "$<$<BOOL:${IRODS_LINUX_DISTRIBUTION_VERSION}>:--target-platform-variant=${IRODS_LINUX_DISTRIBUTION_VERSION}>"
    "--cmake-install-prefix=${CMAKE_INSTALL_PREFIX}"
    "--cmake-install-bindir=${CMAKE_INSTALL_BINDIR}"
    "--cmake-install-sbindir=${CMAKE_INSTALL_SBINDIR}"
    "--cmake-install-libdir=${CMAKE_INSTALL_LIBDIR}"
    "--irods-package-prefix=${IRODS_PACKAGE_PREFIX}"
    "--irods-install-prefix=${IRODS_INSTALL_PREFIX}"
    "--irods-install-bindir=${IRODS_INSTALL_BINDIR}"
    "--irods-install-sbindir=${IRODS_INSTALL_SBINDIR}"
    "--irods-install-libdir=${IRODS_INSTALL_LIBDIR}"
    "--irods-home-dir=${IRODS_HOME_DIRECTORY}"
    "--irods-pluginsdir=${IRODS_PLUGINS_DIRECTORY}"
    "$<$<BOOL:${CMAKE_SHARED_LIBRARY_SUFFIX}>:--sharedlib-suffix=${CMAKE_SHARED_LIBRARY_SUFFIX}>"
    "$<$<BOOL:${CMAKE_EXTRA_SHARED_LIBRARY_SUFFIXES}>:--sharedlib-suffix=$<JOIN:${CMAKE_EXTRA_SHARED_LIBRARY_SUFFIXES},;--sharedlib-suffix=>>"
    "$<$<BOOL:${IRODS_RUNTIME_LIBRARY_PATHS}>:--irods-runtime-lib=$<JOIN:${IRODS_RUNTIME_LIBRARY_PATHS},;--irods-runtime-lib=>>"
    "$<$<BOOL:${LIBCXX_LIBRARIES}>:--libcxx-libpath=$<JOIN:${LIBCXX_LIBRARIES},;--libcxx-libpath=>>"
    "--build-dir=${CMAKE_CURRENT_BINARY_DIR}"
    "--work-dir=${CMAKE_CURRENT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/userspace-tarball"
    "--cmake-tmpdir=${CMAKE_CURRENT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/CMakeTmp"
    "--install-component=icommands"
    "$<$<BOOL:${IRODS_CLIENT_ICOMMANDS_SCRIPTS}>:--script-icommand=$<JOIN:${IRODS_CLIENT_ICOMMANDS_SCRIPTS},;--script-icommand=>>"
    "$<$<BOOL:${ICOMMANDS_USERSPACE_TARBALL_ROOT_DIRNAME}>:--tarball-root-dir-name=${ICOMMANDS_USERSPACE_TARBALL_ROOT_DIRNAME}>"
    "--output=${ICOMMANDS_USERSPACE_TARBALL_PATH}"
  DEPENDS icommands "${ICOMMANDS_USERSPACE_TARBALL_INSTRUMENTATION}"
  OUTPUT "${ICOMMANDS_USERSPACE_TARBALL_PATH}"
  JOB_POOL console
  COMMAND_EXPAND_LISTS
  )
add_custom_target(userspace-tarball DEPENDS "${ICOMMANDS_USERSPACE_TARBALL_PATH}")

include(IrodsCPackCommon)

set(CPACK_INCLUDE_TOPLEVEL_DIRECTORY OFF)
set(CPACK_COMPONENT_INCLUDE_TOPLEVEL_DIRECTORY OFF)
set(CPACK_COMPONENTS_GROUPING IGNORE)
set(CPACK_PACKAGE_VERSION ${IRODS_VERSION})
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "The integrated Rule-Oriented Data System")

set(CPACK_DEB_COMPONENT_INSTALL OFF)
set(CPACK_DEBIAN_PACKAGE_SHLIBDEPS OFF)
set(CPACK_DEBIAN_PACKAGE_CONTROL_STRICT_PERMISSION ON)
set(CPACK_DEBIAN_FILE_NAME DEB-DEFAULT)

set(CPACK_RPM_COMPONENT_INSTALL OFF)
set(CPACK_RPM_PACKAGE_LICENSE "BSD-3-Clause")
set(CPACK_RPM_PACKAGE_AUTOREQ 0)
set(CPACK_RPM_PACKAGE_AUTOPROV 0)
set(CPACK_RPM_PACKAGE_RELOCATABLE ON)
set(CPACK_RPM_FILE_NAME RPM-DEFAULT)
set(CPACK_RPM_POST_INSTALL_SCRIPT_FILE "${CMAKE_CURRENT_SOURCE_DIR}/packaging/postinst")

set(CPACK_ARCHIVE_COMPONENT_INSTALL OFF)

set(CPACK_DEBIAN_PACKAGE_NAME "irods-icommands")
if (OPENSSL_VERSION VERSION_LESS "3.0.0")
  set(CPACK_DEBIAN_PACKAGE_DEPENDS "${IRODS_PACKAGE_DEPENDENCIES_STRING}, irods-runtime (= ${IRODS_VERSION}), libc6, libssl1.1")
else()
  set(CPACK_DEBIAN_PACKAGE_DEPENDS "${IRODS_PACKAGE_DEPENDENCIES_STRING}, irods-runtime (= ${IRODS_VERSION}), libc6, libssl3")
endif()
set(CPACK_DEBIAN_PACKAGE_PROVIDES "irods-icommands (= ${IRODS_VERSION})")
set(CPACK_DEBIAN_PACKAGE_REPLACES "irods-icat, irods-resource")
set(CPACK_DEBIAN_PACKAGE_CONFLICTS "irods-experimental-api-plugin-genquery2")
set(CPACK_DEBIAN_PACKAGE_CONTROL_EXTRA "${CMAKE_CURRENT_SOURCE_DIR}/packaging/postinst;")

set(CPACK_RPM_PACKAGE_NAME "irods-icommands")
set(CPACK_RPM_PACKAGE_OBSOLETES "irods-icat, irods-resource")
set(CPACK_RPM_PACKAGE_CONFLICTS "irods-experimental-api-plugin-genquery2")
set(CPACK_RPM_PACKAGE_REQUIRES "${IRODS_PACKAGE_DEPENDENCIES_STRING}, irods-runtime = ${IRODS_VERSION}")
if (IRODS_LINUX_DISTRIBUTION_NAME STREQUAL "opensuse")
  set(CPACK_RPM_PACKAGE_REQUIRES "${CPACK_RPM_PACKAGE_REQUIRES}, libopenssl1_0_0")
else()
  set(CPACK_RPM_PACKAGE_REQUIRES "${CPACK_RPM_PACKAGE_REQUIRES}, openssl")
endif()

if (NOT CPACK_GENERATOR)
  set(CPACK_GENERATOR ${IRODS_CPACK_GENERATOR} CACHE STRING "CPack generator to use, e.g. {DEB, RPM, TGZ}." FORCE)
  message(STATUS "Setting unspecified CPACK_GENERATOR to ${CPACK_GENERATOR}. This is the correct setting for normal builds.")
endif()

include(CPack)
