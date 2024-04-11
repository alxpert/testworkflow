cmake_minimum_required(VERSION 3.14)
project(flo-ac VERSION 0.6.0)
include(CTest)
include(CMakeDependentOption)
set(CMAKE_C_STANDARD 99)
set(CMAKE_CXX_STANDARD 17)

# TODO should be here, but some submodules are not ready
# TODO remove when all submodules are ready
#add_compile_options(-Wall -Werror -Wno-error=deprecated-declarations)

include(GNUInstallDirs)

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_BINDIR})
set(SYSTEMD_SERVICE_INSTALL_DIRECTORY "${CMAKE_INSTALL_LIBDIR}/systemd/system" CACHE PATH "Path where systemd service files are installed")

set(FLO_AC_VERSION ${CMAKE_PROJECT_VERSION})

# Replace some default flags that cmake set that increase the binaries size
if(CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX)
    string(REPLACE "-rdynamic" "" CMAKE_SHARED_LIBRARY_LINK_CXX_FLAGS "${CMAKE_SHARED_LIBRARY_LINK_CXX_FLAGS}")
    string(REPLACE "-rdynamic" "" CMAKE_SHARED_LIBRARY_LINK_C_FLAGS "${CMAKE_SHARED_LIBRARY_LINK_C_FLAGS}")
endif()
set(CMAKE_SKIP_INSTALL_ALL_DEPENDENCY true)
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${PROJECT_SOURCE_DIR}/cmake")


option(FLOAC_VIRTUAL_BUILD "Build for virtual hardware" OFF)
option(FLOAC_DEV_BUILD "Developper build (install optionnal app)" OFF)
option(FLOAC_LOCAL_BUILD "Indicate this is a local build, not done by CI. Will append the floAC version and git hash to the system version" OFF)
option(FLOAC_YOCTO_BUILD "Indicate if build is running inside Yocto environnement" OFF)
option(FLOAC_ENABLE_VECTOR  "Enable support for ISO-15118 on CCS via Vector's library vSECClib-ccs" ON)
option(FLOAC_ENABLE_RUST "Enable library containing Rust code" ON)
option(FLOAC_BUILD_SHARED_LIBS "Build shared libraries" ON)
option(FLOAC_USE_PRECOMPILED_RTU_RUST "Use the precompiled rtu-rs lib (for Yocto)" OFF)
option(FLOAC_BUILD_DEVICEMODELVIEWER "Build the devicemodelviewer" ON)

# Include CMake Package Manager and use CPMLockfile
include(CPM)
include(cmake/CPM.cmake)
CPMUsePackageLock(package-lock.cmake)

if(FLOAC_BUILD_SHARED_LIBS)
    set(LIBAE_LIBRARY_TYPE SHARED )
else()
    set(LIBAE_LIBRARY_TYPE STATIC )
endif()

set(SYSTEMD_SUPPORT "OFF")
find_file(SYSTEMD_AVAILABLE systemd/sd-daemon.h)
if (SYSTEMD_AVAILABLE)
    set(SYSTEMD_SUPPORT "ON")
endif()

set(UBOOT_SUPPORT "OFF")
find_file(UBOOT_AVAILABLE libuboot.h)
if (UBOOT_AVAILABLE)
    set(UBOOT_SUPPORT "ON")
endif()

#============================================
# Use the flocore lib
#============================================
CPMGetPackage(floCore)

if (floCore_ADDED)
    # Add flocore cmake file to cmake search path
    set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${floCore_SOURCE_DIR}/cmake")
endif()

CPMGetPackage(floACDC)


# TODO move to externals
CPMAddPackage(
    NAME CLI11
    GIT_REPOSITORY https://github.com/CLIUtils/CLI11.git
    GIT_TAG v2.3.2
)

# Not sure why this is required when it's already done in flocore...
find_package(Python COMPONENTS Interpreter REQUIRED)

#============================================
# Patch for protobuf
# Somehow when we cross compile, some projects have no visibility on protobuf::protoc and it create bad make file
# Here we force this target to exist at global level
find_package(protobuf CONFIG REQUIRED)

add_executable(protoc_local IMPORTED GLOBAL)
set_target_properties(protoc_local PROPERTIES IMPORTED_LOCATION ${Protobuf_PROTOC_EXECUTABLE} )
add_executable(protobuf::protoc ALIAS protoc_local)

#============================================

set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# Define package for all app
add_definitions(-DADD_VERSION_MAJOR=${PROJECT_VERSION_MAJOR})
add_definitions(-DADD_VERSION_MINOR=${PROJECT_VERSION_MINOR})
add_definitions(-DADD_VERSION_REV=${PROJECT_VERSION_PATCH})
add_definitions(-DADD_COMMIT_ID=${COMMIT_ID})
add_definitions(-DADD_BUILD_NUMBER=${BUILD_NUMBER})


configure_file(${CMAKE_CURRENT_SOURCE_DIR}/package.sh.in ${CMAKE_BINARY_DIR}/package.sh @ONLY)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/make-sandbox-and-deploy.sh.in ${CMAKE_BINARY_DIR}/make-sandbox-and-deploy.sh @ONLY)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/make-rpm-and-deploy.sh.in ${CMAKE_BINARY_DIR}/make-rpm-and-deploy.sh @ONLY)


set(libae-rtu_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/libae-rtu)
add_subdirectory(lib-rtu-host) # Must be done after importing libae-rtu as we use a symbol defined by libae-rtu

add_subdirectory(libfloac-devices)
add_subdirectory(controller)
add_subdirectory(certif)
add_subdirectory(services)
add_subdirectory(scripts)

# Python apps
add_subdirectory(floac-python-lib)
add_subdirectory(static-config)
add_subdirectory(manufacturing)
add_subdirectory(onboarding)

# Tools
add_subdirectory(tools)


set(CPACK_GENERATOR "RPM")
set(CPACK_PACKAGE_DESCRIPTION ${CMAKE_PROJECT_NAME})
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "Applications for Flo AC products")
set(CPACK_PACKAGE_VENDOR "Flo")
set(CPACK_PACKAGE_VERSION_MAJOR ${CMAKE_PROJECT_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR ${CMAKE_PROJECT_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH ${CMAKE_PROJECT_VERSION_PATCH})
set(CPACK_RPM_EXCLUDE_FROM_AUTO_FILELIST_ADDITION "/lib" "/lib/systemd" "/lib/systemd/system")
set(CPACK_RPM_PRE_INSTALL_SCRIPT_FILE "${CMAKE_CURRENT_SOURCE_DIR}/scripts/rpm-starbuck-pre-install.sh")
set(CPACK_RPM_POST_INSTALL_SCRIPT_FILE "${CMAKE_CURRENT_SOURCE_DIR}/scripts/rpm-starbuck-post-install.sh")
if(DEFINED GIT_REPO_COMMIT_HASH)
    set(CPACK_RPM_PACKAGE_RELEASE ${GIT_REPO_COMMIT_HASH})
else()
    set(CPACK_RPM_PACKAGE_RELEASE "dev")
endif()
set(CPACK_PACKAGE_FILE_NAME "${CMAKE_PROJECT_NAME}-${CMAKE_PROJECT_VERSION}-${CPACK_RPM_PACKAGE_RELEASE}.${CPACK_RPM_PACKAGE_ARCHITECTURE}")
set(CPACK_STRIP_FILES ON)

include(CPack)

