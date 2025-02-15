# Copyright (C) 2013-2020 Bayerische Motoren Werke Aktiengesellschaft (BMW AG)
# This Source Code Form is subject to the terms of the Mozilla Public
# License, v. 2.0. If a copy of the MPL was not distributed with this
# file, You can obtain one at http://mozilla.org/MPL/2.0/.

cmake_minimum_required (VERSION 2.8)

project (CommonAPI-SomeIP)

set(CMAKE_CXX_STANDARD 20)
SET(CMAKE_SKIP_BUILD_RPATH  FALSE)
SET(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE)
SET(CMAKE_INSTALL_RPATH "${ORIGIN}")


# boost using conan 
# Download automatically, you can also just copy the conan.cmake file
if(NOT EXISTS "${CMAKE_BINARY_DIR}/conan.cmake")
   message(STATUS "Downloading conan.cmake from https://github.com/conan-io/cmake-conan")
   file(DOWNLOAD "https://raw.githubusercontent.com/conan-io/cmake-conan/master/conan.cmake"
                  "${CMAKE_BINARY_DIR}/conan.cmake")
endif()

include(${CMAKE_BINARY_DIR}/conan.cmake)

conan_cmake_configure(REQUIRES VSOMEIP/3.0.0.9
                CommonAPI/3.2.0.5
                GENERATORS cmake
                )
conan_cmake_autodetect(settings)

conan_cmake_install(PATH_OR_REFERENCE .
                    BUILD missing
                    SETTINGS ${settings})

include(${CMAKE_BINARY_DIR}/conanbuildinfo.cmake)
conan_basic_setup(TARGETS)

set (CMAKE_VERBOSE_MAKEFILE off)

set (LIBCOMMONAPI_SOMEIP_MAJOR_VERSION 3)
set (LIBCOMMONAPI_SOMEIP_MINOR_VERSION 2)
set (LIBCOMMONAPI_SOMEIP_PATCH_VERSION 0)

message(STATUS "Project name: ${PROJECT_NAME}")

set (COMPONENT_VERSION ${LIBCOMMONAPI_SOMEIP_MAJOR_VERSION}.${LIBCOMMONAPI_SOMEIP_MINOR_VERSION}.${LIBCOMMONAPI_SOMEIP_PATCH_VERSION})
set (COMMONAPI_API_HEADER_VERSION ${LIBCOMMONAPI_SOMEIP_MAJOR_VERSION}.${LIBCOMMONAPI_SOMEIP_MINOR_VERSION})

SET(PACKAGE_VERSION "${COMPONENT_VERSION}") # used in *.cmake.in

OPTION(USE_FILE "Set to OFF to disable file logging" OFF )
message(STATUS "USE_FILE is set to value: ${USE_FILE}")

OPTION(USE_CONSOLE "Set to OFF to disable console logging" OFF )
message(STATUS "USE_CONSOLE is set to value: ${USE_CONSOLE}")

SET(DEFAULT_SEND_TIMEOUT "5000" CACHE STRING "default send timeout")
message(STATUS "DEFAULT_SEND_TIMEOUT is set to value: ${DEFAULT_SEND_TIMEOUT} ms")

IF(USE_FILE)
  add_definitions(-DUSE_FILE)
ENDIF(USE_FILE)
IF(USE_CONSOLE)
  add_definitions(-DUSE_CONSOLE)
ENDIF(USE_CONSOLE)

add_definitions(-DDEFAULT_SEND_TIMEOUT=${DEFAULT_SEND_TIMEOUT})

# Set a default build type if none was specified
set(default_build_type "RelWithDebInfo")
if(NOT CMAKE_BUILD_TYPE)
  message(STATUS "Setting build type to '${default_build_type}' as none was specified.")
  set(CMAKE_BUILD_TYPE "${default_build_type}" CACHE STRING "Choose the type of build." FORCE)
  # Set the possible values of build type for cmake-gui
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()

SET(MAX_LOG_LEVEL "DEBUG" CACHE STRING "maximum log level")
message(STATUS "MAX_LOG_LEVEL is set to value: ${MAX_LOG_LEVEL}")

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DCOMMONAPI_INTERNAL_COMPILATION")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DCOMMONAPI_SOMEIP_VERSION_MAJOR=${LIBCOMMONAPI_SOMEIP_MAJOR_VERSION}")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DCOMMONAPI_SOMEIP_VERSION_MINOR=${LIBCOMMONAPI_SOMEIP_MINOR_VERSION}")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DCOMMONAPI_LOGLEVEL=COMMONAPI_LOGLEVEL_${MAX_LOG_LEVEL}")

if (ENABLE_ADDRESS_ALIASES)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DCOMMONAPI_ENABLE_ADDRESS_ALIASES")
message(STATUS "Enabled address aliasing.")
endif ()

# Package config module not found message macro
macro (pkg_config_module_not_found_message PKG_CONFIG_MODULE)
    message (FATAL_ERROR "pkg-config could not find the required module ${PKG_CONFIG_MODULE}!"
      " Please adjust your PKG_CONFIG_PATH environment variable accordingly.")
endmacro ()

###################################################################################################
# see http://www.cmake.org/Wiki/CMake/Tutorials/How_to_create_a_ProjectConfig.cmake_file
###################################################################################################

# Offer the user the choice of overriding the installation directories
set (INSTALL_LIB_DIR lib CACHE PATH "Installation directory for libraries")
set (INSTALL_BIN_DIR bin CACHE PATH "Installation directory for executables")
set (INSTALL_INCLUDE_DIR include CACHE PATH "Installation directory for header files")

if (WIN32 AND NOT CYGWIN)
    set (DEF_INSTALL_CMAKE_DIR cmake)
else ()
    set (DEF_INSTALL_CMAKE_DIR lib/cmake/CommonAPI-SomeIP-${COMPONENT_VERSION})
endif ()

set (INSTALL_CMAKE_DIR ${DEF_INSTALL_CMAKE_DIR} CACHE PATH "Installation directory for CMake files")

# Make relative paths absolute (needed later on)
foreach (p LIB BIN INCLUDE CMAKE)
    set (var INSTALL_${p}_DIR)
    if (NOT IS_ABSOLUTE "${${var}}")
        set (ABSOLUTE_${var} "${CMAKE_INSTALL_PREFIX}/${${var}}")
    endif ()
endforeach ()

###################################################################################################
include_directories (
    include
    ${CONAN_INCLUDE_DIRS_BOOST}
    ${CONAN_INCLUDE_DIRS_COMMONAPI}
    ${CONAN_INCLUDE_DIRS_VSOMEIP}
)

file (GLOB CommonAPI-SomeIP_SRC "src/CommonAPI/SomeIP/*.cpp")
list (SORT CommonAPI-SomeIP_SRC)

# CommonAPI
add_library (CommonAPI-SomeIP SHARED ${CommonAPI-SomeIP_SRC})
set_target_properties (CommonAPI-SomeIP PROPERTIES VERSION ${COMPONENT_VERSION} SOVERSION ${COMPONENT_VERSION})
target_link_libraries (CommonAPI-SomeIP CONAN_PKG::CommonAPI CONAN_PKG::VSOMEIP)

if (MSVC)
    target_link_libraries(CommonAPI-SomeIP ws2_32 Rpcrt4)
endif()

###################################################################################################

file (GLOB_RECURSE CommonAPI-SomeIP_INCLUDE_INSTALL_FILES "include/*.hpp")
list (SORT CommonAPI-SomeIP_INCLUDE_INSTALL_FILES)

set_target_properties (CommonAPI-SomeIP PROPERTIES PUBLIC_HEADER "${CommonAPI-SomeIP_INCLUDE_INSTALL_FILES}")

install (
    TARGETS CommonAPI-SomeIP
    # IMPORTANT: Add the CommonAPI-SomeIP library to the "export-set"
    EXPORT CommonAPI-SomeIPTargets
	LIBRARY DESTINATION ${INSTALL_LIB_DIR}
	RUNTIME DESTINATION ${INSTALL_BIN_DIR}
	ARCHIVE DESTINATION ${INSTALL_LIB_DIR}
    PUBLIC_HEADER DESTINATION "${INSTALL_INCLUDE_DIR}/CommonAPI/SomeIP"
)

##############################################################################
# exporting, configuring and installing of cmake files

# Add all targets to the build-tree export set
export (TARGETS CommonAPI-SomeIP
        FILE "${PROJECT_BINARY_DIR}/CommonAPI-SomeIPTargets.cmake")

# Export the package for use from the build-tree
# (this registers the build-tree with a global CMake-registry)
export (PACKAGE CommonAPI-SomeIP)

# Create the CommonAPI-SomeIPConfig.cmake and CommonAPI-SomeIPConfigVersion files
file (RELATIVE_PATH REL_INCLUDE_DIR "${ABSOLUTE_INSTALL_CMAKE_DIR}" "${ABSOLUTE_INSTALL_INCLUDE_DIR}")

# ... for the build tree
set (CONF_INCLUDE_DIRS "${PROJECT_SOURCE_DIR}/include")
configure_file (${CMAKE_CURRENT_SOURCE_DIR}/cmake/CommonAPI-SomeIPConfig.cmake.in
    "${PROJECT_BINARY_DIR}/CommonAPI-SomeIPConfig.cmake" @ONLY)

# ... for the install tree
set (CONF_INCLUDE_DIRS "\${COMMONAPI_SOMEIP_CMAKE_DIR}/${REL_INCLUDE_DIR}")
configure_file (${CMAKE_CURRENT_SOURCE_DIR}/cmake/CommonAPI-SomeIPConfig.cmake.in
    "${PROJECT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/CommonAPI-SomeIPConfig.cmake" @ONLY)

# ... for both
configure_file (${CMAKE_CURRENT_SOURCE_DIR}/cmake/CommonAPI-SomeIPConfigVersion.cmake.in
    "${PROJECT_BINARY_DIR}/CommonAPI-SomeIPConfigVersion.cmake" @ONLY)

# Install the CommonAPI-SomeIPConfig.cmake and CommonAPI-SomeIPConfigVersion.cmake
install (
    FILES
    "${PROJECT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/CommonAPI-SomeIPConfig.cmake"
    "${PROJECT_BINARY_DIR}/CommonAPI-SomeIPConfigVersion.cmake"
    DESTINATION "${INSTALL_CMAKE_DIR}"
)

# Install the export set for use with the install-tree
install (
    EXPORT CommonAPI-SomeIPTargets
    DESTINATION "${INSTALL_CMAKE_DIR}"
)

##############################################################################
# create pkg-config file
if(NOT WIN32)
        configure_file(CommonAPI-SomeIP.pc.in ${PROJECT_BINARY_DIR}/CommonAPI-SomeIP.pc @ONLY)
        install(FILES ${PROJECT_BINARY_DIR}/CommonAPI-SomeIP.pc DESTINATION lib/pkgconfig)
endif()

##############################################################################
# maintainer-clean
add_custom_target(maintainer-clean COMMAND rm -rf *)
