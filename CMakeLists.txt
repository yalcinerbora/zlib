cmake_minimum_required(VERSION 3.23)
set(CMAKE_ALLOW_LOOSE_LOOP_CONSTRUCTS ON)

project(zlib C)

set(ZLIB_VERSION "1.2.13")

include(CheckTypeSize)
include(CheckFunctionExists)
include(CheckIncludeFile)
include(CheckCSourceCompiles)
include(GNUInstallDirs)
enable_testing()

option(ZLIB_BUILD_SHARED "Build as shared library" ON)
option(ZLIB_BUILD_EXAMPLES "Build the examples" OFF)
option(ZLIB_INSTALL "Generate install target" OFF)

check_include_file(sys/types.h HAVE_SYS_TYPES_H)
check_include_file(stdint.h    HAVE_STDINT_H)
check_include_file(stddef.h    HAVE_STDDEF_H)

#
# Check to see if we have large file support
#
set(CMAKE_REQUIRED_DEFINITIONS -D_LARGEFILE64_SOURCE=1)
# We add these other definitions here because CheckTypeSize.cmake
# in CMake 2.4.x does not automatically do so and we want
# compatibility with CMake 2.4.x.
if(HAVE_SYS_TYPES_H)
    list(APPEND CMAKE_REQUIRED_DEFINITIONS -DHAVE_SYS_TYPES_H)
endif()
if(HAVE_STDINT_H)
    list(APPEND CMAKE_REQUIRED_DEFINITIONS -DHAVE_STDINT_H)
endif()
if(HAVE_STDDEF_H)
    list(APPEND CMAKE_REQUIRED_DEFINITIONS -DHAVE_STDDEF_H)
endif()
check_type_size(off64_t OFF64_T)
if(HAVE_OFF64_T)
   add_definitions(-D_LARGEFILE64_SOURCE=1)
endif()
set(CMAKE_REQUIRED_DEFINITIONS) # clear variable

#
# Check for fseeko
#
check_function_exists(fseeko HAVE_FSEEKO)
if(NOT HAVE_FSEEKO)
    add_definitions(-DNO_FSEEKO)
endif()

#
# Check for unistd.h
#
check_include_file(unistd.h Z_HAVE_UNISTD_H)

if(MSVC)
    set(CMAKE_DEBUG_POSTFIX "d")
    add_definitions(-D_CRT_SECURE_NO_DEPRECATE)
    add_definitions(-D_CRT_NONSTDC_NO_DEPRECATE)
    include_directories(${CMAKE_CURRENT_SOURCE_DIR})
endif()

if(NOT CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_CURRENT_BINARY_DIR)
    # If we're doing an out of source build and the user has a zconf.h
    # in their source tree...
    if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/zconf.h)
        message(STATUS "Renaming")
        message(STATUS "    ${CMAKE_CURRENT_SOURCE_DIR}/zconf.h")
        message(STATUS "to 'zconf.h.included' because this file is included with zlib")
        message(STATUS "but CMake generates it automatically in the build directory.")
        file(RENAME ${CMAKE_CURRENT_SOURCE_DIR}/zconf.h ${CMAKE_CURRENT_SOURCE_DIR}/zconf.h.included)
  endif()
endif()

set(ZLIB_PC ${CMAKE_CURRENT_BINARY_DIR}/zlib.pc)
configure_file( ${CMAKE_CURRENT_SOURCE_DIR}/zlib.pc.cmakein
		${ZLIB_PC} @ONLY)
configure_file(	${CMAKE_CURRENT_SOURCE_DIR}/zconf.h.cmakein
		${CMAKE_CURRENT_BINARY_DIR}/zconf.h @ONLY)
include_directories(${CMAKE_CURRENT_BINARY_DIR} ${CMAKE_SOURCE_DIR})


#============================================================================
# zlib
#============================================================================

set(ZLIB_PUBLIC_HDRS
    zlib.h
)

set(ZLIB_GENERATED_HDRS
    ${CMAKE_CURRENT_BINARY_DIR}/zconf.h)

set(ZLIB_PRIVATE_HDRS
    crc32.h
    deflate.h
    gzguts.h
    inffast.h
    inffixed.h
    inflate.h
    inftrees.h
    trees.h
    zutil.h
)
set(ZLIB_SRCS
    adler32.c
    compress.c
    crc32.c
    deflate.c
    gzclose.c
    gzlib.c
    gzread.c
    gzwrite.c
    inflate.c
    infback.c
    inftrees.c
    inffast.c
    trees.c
    uncompr.c
    zutil.c
)

if(NOT MINGW)
    set(ZLIB_DLL_SRCS
        win32/zlib1.rc # If present will override custom build rule below.
    )
endif()

# parse the full version number from zlib.h and include in ZLIB_FULL_VERSION
file(READ ${CMAKE_CURRENT_SOURCE_DIR}/zlib.h _zlib_h_contents)
string(REGEX REPLACE ".*#define[ \t]+ZLIB_VERSION[ \t]+\"([-0-9A-Za-z.]+)\".*"
    "\\1" ZLIB_FULL_VERSION ${_zlib_h_contents})

if(MINGW)
    # This gets us DLL resource information when compiling on MinGW.
    if(NOT CMAKE_RC_COMPILER)
        set(CMAKE_RC_COMPILER windres.exe)
    endif()

    add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/zlib1rc.obj
                       COMMAND ${CMAKE_RC_COMPILER}
                            -D GCC_WINDRES
                            -I ${CMAKE_CURRENT_SOURCE_DIR}
                            -I ${CMAKE_CURRENT_BINARY_DIR}
                            -o ${CMAKE_CURRENT_BINARY_DIR}/zlib1rc.obj
                            -i ${CMAKE_CURRENT_SOURCE_DIR}/win32/zlib1.rc)
    set(ZLIB_DLL_SRCS ${CMAKE_CURRENT_BINARY_DIR}/zlib1rc.obj)
endif(MINGW)

if(ZLIB_BUILD_SHARED)
    add_library(ZLIB SHARED)
    target_sources(ZLIB PRIVATE ${ZLIB_DLL_SRCS})

    set_target_properties(ZLIB PROPERTIES DEFINE_SYMBOL ZLIB_DLL)
    set_target_properties(ZLIB PROPERTIES SOVERSION 1)
    if(NOT CYGWIN)
        # This property causes shared libraries on Linux to have the full version
        # encoded into their final filename.  We disable this on Cygwin because
        # it causes cygz-${ZLIB_FULL_VERSION}.dll to be created when cygz.dll
        # seems to be the default.
        #
        # This has no effect with MSVC, on that platform the version info for
        # the DLL comes from the resource file win32/zlib1.rc
        set_target_properties(ZLIB PROPERTIES VERSION ${ZLIB_FULL_VERSION})
    endif()

    if(UNIX AND NOT APPLE)
        set_target_properties(ZLIB PROPERTIES LINK_FLAGS "-Wl,--version-script,\"${CMAKE_CURRENT_SOURCE_DIR}/zlib.map\"")
    endif()

else()
    add_library(ZLIB STATIC)
endif()

target_sources(ZLIB PRIVATE ${ZLIB_SRCS})
target_sources(ZLIB
                PUBLIC FILE_SET HEADERS
                BASE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}
                FILES ${ZLIB_PUBLIC_HDRS})
target_sources(ZLIB PRIVATE ${ZLIB_PRIVATE_HDRS})
target_sources(ZLIB PRIVATE ${ZLIB_GENERATED_HDRS})


if(UNIX)
    # On unix-like platforms the library is almost always called libz
   set_target_properties(ZLIB PROPERTIES OUTPUT_NAME z)
elseif(ZLIB_BUILD_SHARED AND WIN32)
    # Creates zlib1.dll when building shared library version
    set_target_properties(ZLIB PROPERTIES SUFFIX "1.dll")
endif()

add_library(ZLIB::ZLIB ALIAS ZLIB)

#============================================================================
# Installation
#============================================================================
if(ZLIB_INSTALL)

    install(TARGETS ZLIB
            EXPORT ZLIBTargets
            DESTINATION ${CMAKE_INSTALL_LIBDIR}
            FILE_SET HEADERS
            DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/zlib)

    install(FILES ${ZLIB_GENERATED_HDRS}
            DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/zlib)

    # From the cmake docs tutorial
    include(CMakePackageConfigHelpers)

    set(ZLIB_CONFIG_DIR ${CMAKE_INSTALL_LIBDIR}/cmake/zlib)

    # generate the config file that includes the exports
    configure_package_config_file(${CMAKE_CURRENT_SOURCE_DIR}/zlibConfig.cmake.in
        "${CMAKE_CURRENT_BINARY_DIR}/ZLIBConfig.cmake"
        INSTALL_DESTINATION ${ZLIB_CONFIG_DIR}
        NO_CHECK_REQUIRED_COMPONENTS_MACRO
    )
    write_basic_package_version_file("${CMAKE_CURRENT_BINARY_DIR}/ZLIBConfigVersion.cmake"
                                     VERSION ${ZLIB_VERSION}
                                     COMPATIBILITY SameMajorVersion)

    install(FILES
            "${CMAKE_CURRENT_BINARY_DIR}/ZLIBConfig.cmake"
            "${CMAKE_CURRENT_BINARY_DIR}/ZLIBConfigVersion.cmake"
            DESTINATION ${ZLIB_CONFIG_DIR})

    install(EXPORT ZLIBTargets
            NAMESPACE ZLIB::
            FILE ZLIBTargets.cmake
            DESTINATION ${ZLIB_CONFIG_DIR})

    # TODO: Probably this is not standart
    install(FILES ${ZLIB_PC} DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
endif()

#============================================================================
# Example binaries
#============================================================================
if(ZLIB_BUILD_EXAMPLES)
    add_executable(example test/example.c)
    target_link_libraries(example zlib)
    add_test(example example)

    add_executable(minigzip test/minigzip.c)
    target_link_libraries(minigzip zlib)

    if(HAVE_OFF64_T)
        add_executable(example64 test/example.c)
        target_link_libraries(example64 zlib)
        set_target_properties(example64 PROPERTIES COMPILE_FLAGS "-D_FILE_OFFSET_BITS=64")
        add_test(example64 example64)

        add_executable(minigzip64 test/minigzip.c)
        target_link_libraries(minigzip64 zlib)
        set_target_properties(minigzip64 PROPERTIES COMPILE_FLAGS "-D_FILE_OFFSET_BITS=64")
    endif()
endif()
