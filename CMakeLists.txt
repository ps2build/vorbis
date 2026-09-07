cmake_minimum_required(VERSION 3.6)

# Set the toolchain file before project() - real, previously-latent bug
# found for real porting Open-PS2-Loader: this repo's own CI never
# passed -DCMAKE_TOOLCHAIN_FILE explicitly and had no default here
# either, so the published release silently configured against the HOST
# compiler (x86 COFF objects in the shipped .a, useless for EE linking)
# instead of the real mips64r5900el-ps2-elf cross-compiler. Same real
# pattern already used correctly by ps2gl/ps2stuff's own CMakeLists.txt.
if(NOT CMAKE_TOOLCHAIN_FILE)
    if(DEFINED ENV{PS2DEV})
        set(CMAKE_TOOLCHAIN_FILE "$ENV{PS2DEV}/cmake/ps2dev.cmake" CACHE FILEPATH "Toolchain file")
    else()
        message(FATAL_ERROR "PS2DEV environment variable is not set. Please set it to your PS2DEV installation path.")
    endif()
endif()

project(vorbis)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

# Required modules
include(CTest)
include(GNUInstallDirs)
include(CheckIncludeFiles)
include(CheckLibraryExists)

# Build options
option(BUILD_SHARED_LIBS "Build shared library" OFF)
if(APPLE)
    option(BUILD_FRAMEWORK "Build Framework bundle for OSX" OFF)
endif()

if(BUILD_FRAMEWORK)
	set(BUILD_SHARED_LIBS TRUE)
endif()

option(INSTALL_CMAKE_PACKAGE_MODULE "Install CMake package configuration module" ON)

# Extract project version from configure.ac
file(READ configure.ac CONFIGURE_AC_CONTENTS)
string(REGEX MATCH "AC_INIT\\(\\[libvorbis\\],\\[([0-9]*).([0-9]*).([0-9]*)" DUMMY ${CONFIGURE_AC_CONTENTS})
set(PROJECT_VERSION_MAJOR ${CMAKE_MATCH_1})
set(PROJECT_VERSION_MINOR ${CMAKE_MATCH_2})
set(PROJECT_VERSION_PATCH ${CMAKE_MATCH_3})
set(PROJECT_VERSION ${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}.${PROJECT_VERSION_PATCH})

# Helper function to get version-info
function(get_version_info result current_var_name age_var_name revision_var_name)
    string(REGEX MATCH "${current_var_name}=([0-9]*)" DUMMY ${CONFIGURE_AC_CONTENTS})
    set(VERSION_INFO_CURRENT ${CMAKE_MATCH_1})

    string(REGEX MATCH "${age_var_name}=([0-9]*)" DUMMY ${CONFIGURE_AC_CONTENTS})
    set(VERSION_INFO_AGE ${CMAKE_MATCH_1})

    string(REGEX MATCH "${revision_var_name}=([0-9]*)" DUMMY ${CONFIGURE_AC_CONTENTS})
    set(VERSION_INFO_REVISION ${CMAKE_MATCH_1})

    math(EXPR VERSION_INFO_CURRENT_MINUS_AGE "${VERSION_INFO_CURRENT} - ${VERSION_INFO_AGE}")

    set(${result} "${VERSION_INFO_CURRENT_MINUS_AGE}.${VERSION_INFO_AGE}.${VERSION_INFO_REVISION}" PARENT_SCOPE)
endfunction()

# Helper function to configure pkg-config files
function(configure_pkg_config_file pkg_config_file_in)
    set(prefix ${CMAKE_INSTALL_PREFIX})
    set(exec_prefix ${CMAKE_INSTALL_FULL_BINDIR})
    set(libdir ${CMAKE_INSTALL_FULL_LIBDIR})
    set(includedir ${CMAKE_INSTALL_FULL_INCLUDEDIR})
    set(VERSION ${PROJECT_VERSION})
    string(REPLACE ".in" "" pkg_config_file ${pkg_config_file_in})
    configure_file(${pkg_config_file_in} ${pkg_config_file} @ONLY)
endfunction()

message(STATUS "Configuring ${PROJECT_NAME} ${PROJECT_VERSION}")

# Find math library

check_library_exists(m floor "" HAVE_LIBM)
if(HAVE_LIBM)
    set(VORBIS_LIBS "-lm")
endif()

# Find ogg dependency
if(NOT TARGET Ogg::ogg)
    find_package(Ogg REQUIRED)
endif()

add_subdirectory(lib)

if(BUILD_TESTING)
    add_subdirectory(test)
endif()

configure_pkg_config_file(vorbis.pc.in)
configure_pkg_config_file(vorbisenc.pc.in)
configure_pkg_config_file(vorbisfile.pc.in)

install(
    FILES
    ${CMAKE_CURRENT_BINARY_DIR}/vorbis.pc
    ${CMAKE_CURRENT_BINARY_DIR}/vorbisenc.pc
    ${CMAKE_CURRENT_BINARY_DIR}/vorbisfile.pc
    DESTINATION
    ${CMAKE_INSTALL_LIBDIR}/pkgconfig
)

# package.yaml - so ps2build's own package tooling can see this package
# like any other installed under packages/<tier>/<name>, even though
# it's built by CMake rather than "ps2build install". Same pattern as
# ps2/libjpeg-turbo.
install(
    FILES ${CMAKE_SOURCE_DIR}/package.yaml
    DESTINATION .
)
