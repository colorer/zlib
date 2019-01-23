cmake_minimum_required(VERSION 3.0.2)
if(POLICY CMP0048)
    cmake_policy(SET CMP0048 NEW)
endif()
set(CMAKE_ALLOW_LOOSE_LOOP_CONSTRUCTS ON)

project(ZLIB VERSION "1.2.11.1" LANGUAGES C)

set(VERSION "${PROJECT_VERSION}")

option(ZLIB_EXAMPLES "Build zlib example programs" ON)
option(ZLIB_MINIZIP "Build contrib/minizip alongside zlib" OFF)
option(ZLIB_BUILD_SHARED_LIBS "Build shared (dynamic) libraries" ON)
option(ZLIB_BUILD_STATIC_LIBS "Build both static libraries" ON)
option(SKIP_INSTALL_ALL "Skip installation of everything" OFF)
option(SKIP_INSTALL_FILES "Skip installation of manual pages and pkgconfig files" OFF)
option(SKIP_INSTALL_HEADERS "Skip installation of headers" OFF)
option(SKIP_INSTALL_LIBRARIES "Skip installation of binaries, libraries, and symbol files" OFF)
option(ZLIB_BUILD_SHARED_RUNTIME "Build with shared linked runtine library (MSVC)" ON)

set(INSTALL_BIN_DIR "${CMAKE_INSTALL_PREFIX}/bin" CACHE PATH "Installation directory for executables")
set(INSTALL_LIB_DIR "${CMAKE_INSTALL_PREFIX}/lib" CACHE PATH "Installation directory for libraries")
set(INSTALL_INC_DIR "${CMAKE_INSTALL_PREFIX}/include" CACHE PATH "Installation directory for headers")
set(INSTALL_MAN_DIR "${CMAKE_INSTALL_PREFIX}/share/man" CACHE PATH "Installation directory for manual pages")
set(INSTALL_PKGCONFIG_DIR "${CMAKE_INSTALL_PREFIX}/share/pkgconfig" CACHE PATH "Installation directory for pkgconfig (.pc) files")

include(CheckTypeSize)
include(CheckFunctionExists)
include(CheckIncludeFile)
include(CheckCSourceCompiles)
enable_testing()

check_include_file(sys/types.h HAVE_SYS_TYPES_H)
check_include_file(stdint.h    HAVE_STDINT_H)
check_include_file(stddef.h    HAVE_STDDEF_H)

if(ZLIB_BUILD_SHARED_LIBS)
    set(BUILD_SHARED_LIBS ON)
endif()

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
endif()

set(ZLIB_PC ${CMAKE_CURRENT_BINARY_DIR}/zlib.pc)
configure_file( ${CMAKE_CURRENT_SOURCE_DIR}/zlib.pc.cmakein
		${ZLIB_PC} @ONLY)
configure_file(	${CMAKE_CURRENT_SOURCE_DIR}/zconf.h.cmakein
		${CMAKE_CURRENT_BINARY_DIR}/zconf.h @ONLY)

#============================================================================
# zlib
#============================================================================

set(ZLIB_PUBLIC_HDRS
    ${CMAKE_CURRENT_BINARY_DIR}/zconf.h
    zlib.h
)
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

if(BUILD_SHARED_LIBS AND NOT MINGW AND NOT MSYS)
    set(ZLIB_DLL_SRCS
        win32/zlib1.rc # If present will override custom build rule below.
    )
endif()

# parse the full version number from zlib.h and include in ZLIB_FULL_VERSION
file(READ ${CMAKE_CURRENT_SOURCE_DIR}/zlib.h _zlib_h_contents)
string(REGEX REPLACE ".*#define[ \t]+ZLIB_VERSION[ \t]+\"([-0-9A-Za-z.]+)\".*"
    "\\1" ZLIB_FULL_VERSION ${_zlib_h_contents})

if(BUILD_SHARED_LIBS AND (MINGW OR CYGWIN))
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
endif()

if(MSVC)
    if(NOT ZLIB_BUILD_SHARED_RUNTIME)
        set(CompilerFlags
            CMAKE_C_FLAGS
            CMAKE_C_FLAGS_DEBUG
            CMAKE_C_FLAGS_RELEASE
            )
        foreach(CompilerFlag ${CompilerFlags})
            string(REPLACE "/MDd" "/MTd" ${CompilerFlag} "${${CompilerFlag}}")
            string(REPLACE "/MD" "/MT" ${CompilerFlag} "${${CompilerFlag}}")
        endforeach()
    endif()
endif()

if(ZLIB_BUILD_SHARED_LIBS)
    add_library(zlib SHARED ${ZLIB_SRCS} ${ZLIB_DLL_SRCS} ${ZLIB_PUBLIC_HDRS} ${ZLIB_PRIVATE_HDRS})
    target_include_directories(zlib PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<INSTALL_INTERFACE:include>)
    set_target_properties(zlib PROPERTIES
        DEFINE_SYMBOL ZLIB_DLL
        SOVERSION ${PROJECT_VERSION_MAJOR})

    if(UNIX)
        # On unix-like platforms the library is almost always called libz
        set_target_properties(zlib PROPERTIES OUTPUT_NAME z)
    endif()
endif()

if(ZLIB_BUILD_STATIC_LIBS)
    # This preserves the original behavior of the zlib cmake build, however combining shared and static
    # builds does not afford customization of compiler options typically used on some platforms for each type
    add_library(zlibstatic STATIC ${ZLIB_SRCS} ${ZLIB_PUBLIC_HDRS} ${ZLIB_PRIVATE_HDRS})
    target_include_directories(zlibstatic PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<INSTALL_INTERFACE:include>)
    if(UNIX)
        # On unix-like platforms the library is almost always called libz
        set_target_properties(zlibstatic PROPERTIES OUTPUT_NAME z)
    endif()
endif()

if(ZLIB_BUILD_SHARED_LIBS)
    add_library(ZLIB::ZLIB ALIAS zlib)
else()
    add_library(ZLIB::ZLIB ALIAS zlibstatic)
endif()

if(ZLIB_BUILD_SHARED_LIBS)
    if(NOT CYGWIN)
        # This property causes shared libraries on Linux to have the full version
        # encoded into their final filename.  We disable this on Cygwin because
        # it causes cygz-${PROJECT_VERSION}.dll to be created when cygz.dll
        # seems to be the default.
        #
        # This has no effect with MSVC, on that platform the version info for
        # the DLL comes from the resource file win32/zlib1.rc
        set_target_properties(zlib PROPERTIES VERSION ${PROJECT_VERSION})
    endif()

    if (UNIX AND NOT APPLE)
        set_target_properties(zlib PROPERTIES LINK_FLAGS "-Wl,--version-script,\"${CMAKE_CURRENT_SOURCE_DIR}/zlib.map\"")
    endif()

    if(WIN32)
        # Creates zlib1.dll when building shared library version
        set_target_properties(zlib PROPERTIES OUTPUT_NAME "zlib1")
    endif()
endif()

if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL)
    if(ZLIB_BUILD_SHARED_LIBS)
        install(TARGETS zlib
            RUNTIME DESTINATION "${INSTALL_BIN_DIR}"
            ARCHIVE DESTINATION "${INSTALL_LIB_DIR}"
            LIBRARY DESTINATION "${INSTALL_LIB_DIR}")
    endif()
    if(ZLIB_BUILD_STATIC_LIBS)
        install(TARGETS zlibstatic
            RUNTIME DESTINATION "${INSTALL_BIN_DIR}"
            ARCHIVE DESTINATION "${INSTALL_LIB_DIR}"
            LIBRARY DESTINATION "${INSTALL_LIB_DIR}")
    endif()
endif()
if(NOT SKIP_INSTALL_HEADERS AND NOT SKIP_INSTALL_ALL )
    install(FILES ${ZLIB_PUBLIC_HDRS} DESTINATION "${INSTALL_INC_DIR}")
endif()
if(NOT SKIP_INSTALL_FILES AND NOT SKIP_INSTALL_ALL )
    install(FILES zlib.3 DESTINATION "${INSTALL_MAN_DIR}/man3")
endif()
if(NOT SKIP_INSTALL_FILES AND NOT SKIP_INSTALL_ALL )
    install(FILES ${ZLIB_PC} DESTINATION "${INSTALL_PKGCONFIG_DIR}")
endif()

#============================================================================
# contrib/minizip
#============================================================================

if(ZLIB_MINIZIP)
    add_library(minizip STATIC
        contrib/minizip/ioapi.c
        contrib/minizip/ioapi.h
        contrib/minizip/unzip.c
        contrib/minizip/unzip.h
        contrib/minizip/zip.c
        contrib/minizip/zip.h
        )
    target_link_libraries(minizip zlibstatic)
    target_include_directories(minizip INTERFACE
        contrib/minizip
        ${CMAKE_CURRENT_SOURCE_DIR}
        ${CMAKE_CURRENT_BINARY_DIR}
        )
    add_library(ZLIB::MINIZIP ALIAS minizip)
endif(ZLIB_MINIZIP)

#============================================================================
# Example binaries
#============================================================================
if(ZLIB_EXAMPLES)
    add_executable(example test/example.c)
    target_link_libraries(example ZLIB::ZLIB)
    add_test(example example)

    add_executable(minigzip test/minigzip.c)
    target_link_libraries(minigzip ZLIB::ZLIB)

    if(HAVE_OFF64_T)
        add_executable(example64 test/example.c)
        target_link_libraries(example64 ZLIB::ZLIB)
        set_target_properties(example64 PROPERTIES COMPILE_FLAGS "-D_FILE_OFFSET_BITS=64")
        add_test(example64 example64)

        add_executable(minigzip64 test/minigzip.c)
        target_link_libraries(minigzip64 ZLIB::ZLIB)
        set_target_properties(minigzip64 PROPERTIES COMPILE_FLAGS "-D_FILE_OFFSET_BITS=64")
    endif()
endif()