# vim: syntax=cmake
if(NOT CMAKE_BUILD_TYPE)
    # default to Release build for GCC builds
    set(CMAKE_BUILD_TYPE Release CACHE STRING
        "Choose the type of build, options are: None(CMAKE_CXX_FLAGS or CMAKE_C_FLAGS used) Debug Release RelWithDebInfo MinSizeRel."
        FORCE)
endif()

project (x265)
cmake_minimum_required (VERSION 2.8.8) # OBJECT libraries require 2.8.8
include(CheckIncludeFiles)
include(CheckFunctionExists)
include(CheckCXXCompilerFlag)

# X265_BUILD must be incremented each time the public API is changed
set(X265_BUILD 4)
configure_file("${PROJECT_SOURCE_DIR}/x265.def.in"
               "${PROJECT_BINARY_DIR}/x265.def")
configure_file("${PROJECT_SOURCE_DIR}/x265_config.h.in"
               "${PROJECT_BINARY_DIR}/x265_config.h")

SET(CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake" "${CMAKE_MODULE_PATH}")

if ("${CMAKE_SIZEOF_VOID_P}" MATCHES 8)
    set(X64 1)
    add_definitions(-DX86_64=1)
endif()

if (CMAKE_GENERATOR STREQUAL "Xcode")
  set(XCODE 1)
endif()
if (APPLE)
  add_definitions(-DMACOS)
endif()

if(${CMAKE_CXX_COMPILER_ID} STREQUAL "Clang")
    set(CLANG 1)
endif()
if(${CMAKE_CXX_COMPILER_ID} STREQUAL "Intel")
    set(INTEL_CXX 1)
endif()
if(${CMAKE_CXX_COMPILER_ID} STREQUAL "GNU")
    set(GCC 1)
endif()

if(INTEL_CXX AND WIN32)
    # treat icl roughly like MSVC
    set(MSVC 1)
endif()
if(MSVC)
    option(STATIC_LINK_CRT "Statically link C runtime for release builds" OFF)
    if (STATIC_LINK_CRT)
        set(CompilerFlags CMAKE_CXX_FLAGS_RELEASE CMAKE_C_FLAGS_RELEASE)
        foreach(CompilerFlag ${CompilerFlags})
            string(REPLACE "/MD" "/MT" ${CompilerFlag} "${${CompilerFlag}}")
        endforeach()
    endif (STATIC_LINK_CRT)
    # Enforce coding standards.  Full warnings and warnings as errors
    add_definitions(/W4 /WX /D_CRT_SECURE_NO_WARNINGS)
    add_definitions(/Ob2) # always inline
    add_definitions(/Oi)  # enable intrinsics
    add_definitions(/MP)  # multithreaded build

    check_include_files(stdint.h HAVE_STDINT_H)
    if(NOT HAVE_STDINT_H)
        include_directories(compat/msvc)
    endif()
endif(MSVC)

if(INTEL_CXX AND UNIX)
    # treat icpc roughly like gcc
    set(GCC 1)
    add_definitions(-Wall -Wextra -Wshadow -no-vec)
elseif(CLANG)
    # treat clang roughly like gcc
    set(GCC 1)
    add_definitions(-Wall -Wextra -Wshadow -ffast-math)
elseif(CMAKE_COMPILER_IS_GNUCXX)
    add_definitions(-Wall -Wextra -Wshadow -ffast-math)
    check_cxx_compiler_flag(-Wno-narrowing GCC_HAS_NO_NARROWING) 
    check_cxx_compiler_flag(-mstackrealign GCC_HAS_STACK_REALIGN) 
    if (GCC_HAS_STACK_REALIGN)
        add_definitions(-mstackrealign)
    endif()
    execute_process(COMMAND ${CMAKE_CXX_COMPILER} -dumpversion OUTPUT_VARIABLE GCC_VERSION)
endif()
if (GCC)
    option(WARNINGS_AS_ERRORS "Stop compiles on first warning" OFF)
    if(WARNINGS_AS_ERRORS)
        add_definitions(-Werror)
    endif(WARNINGS_AS_ERRORS)
    if(X64 AND NOT WIN32)
        add_definitions(-fPIC)
    endif(X64 AND NOT WIN32)
endif(GCC)

option(HIGH_BIT_DEPTH "Use 16bit pixels internally" OFF)
if(HIGH_BIT_DEPTH)
    add_definitions(-DHIGH_BIT_DEPTH=1)
else(HIGH_BIT_DEPTH)
    add_definitions(-DHIGH_BIT_DEPTH=0)
endif(HIGH_BIT_DEPTH)

option(LOG_CU_STATISTICS "Log Mode Decision Statistics at the CU Level" OFF)
if(LOG_CU_STATISTICS)
    add_definitions(-DLOG_CU_STATISTICS=1)
else(LOG_CU_STATISTICS)
    add_definitions(-DLOG_CU_STATISTICS=0)
endif(LOG_CU_STATISTICS)

find_package(Yasm)
if(YASM_FOUND)
    if (YASM_VERSION_STRING VERSION_LESS "1.2.0")
        message(STATUS "Yasm version ${YASM_VERSION_STRING} is too old. 1.2.0 or later required")
        option(ENABLE_ASSEMBLY "Enable use of assembly coded primitives" OFF)
    else()
        message(STATUS "Found Yasm ${YASM_VERSION_STRING} to build assembly primitives")
        option(ENABLE_ASSEMBLY "Enable use of assembly coded primitives" ON)
    endif()
endif(YASM_FOUND)

if(UNIX)
    SET(PLATFORM_LIBS pthread)
    if(NOT ${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
        SET(PLATFORM_LIBS ${PLATFORM_LIBS} rt)
    endif(NOT ${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
endif(UNIX)

option(ENABLE_PPA "Enable PPA profiling instrumentation" OFF)
if(ENABLE_PPA)
    add_definitions(-DENABLE_PPA)
    add_subdirectory(PPA)
    SET(PLATFORM_LIBS ${PLATFORM_LIBS} PPA)
    if(UNIX)
        SET(PLATFORM_LIBS ${PLATFORM_LIBS} dl)
    endif(UNIX)
endif(ENABLE_PPA)

if (WIN32)
    # Visual leak detector
    find_package(VLD)
    if(VLD_FOUND)
        add_definitions(-DHAVE_VLD)
        include_directories(${VLD_INCLUDE_DIRS})
        set(PLATFORM_LIBS ${PLATFORM_LIBS} ${VLD_LIBRARIES})
        link_directories(${VLD_LIBRARY_DIRS})
    endif()
    option(WINXP_SUPPORT "Make binaries compatible with Windows XP" OFF)
    if(WINXP_SUPPORT)
        # force workarounds for atomic intrinsics introduced after XP
        add_definitions(-D_WIN32_WINNT=_WIN32_WINNT_WINXP)
    endif(WINXP_SUPPORT)
endif()

include(version) # determine X265_VERSION and X265_LATEST_TAG
include_directories(. Lib common encoder "${PROJECT_BINARY_DIR}")
add_subdirectory(common)
add_subdirectory(encoder)

if((MSVC_IDE OR XCODE) AND ENABLE_ASSEMBLY)
    # this is horrible. ugly, and hacky, and it reproduces logic found
    # in the yasm CMake modules, but this is required because of this cmake bug
    # http://www.cmake.org/Bug/print_bug_page.php?bug_id=8170
    if (X64)
        if(APPLE)
            set(FLAGS -f macho64 -m amd64 -DPREFIX -DPIC -DARCH_X86_64=1 -DHAVE_ALIGNED_STACK=1)
        else()
            set(FLAGS -f win64 -m amd64 -DARCH_X86_64=1 -DHAVE_ALIGNED_STACK=0)
        endif()
    else()
        if(APPLE)
            set(FLAGS -f macho -DARCH_X86_64=0 -DHAVE_ALIGNED_STACK=1 -DPREFIX)
        else()
            set(FLAGS -f win32 -DARCH_X86_64=0 -DHAVE_ALIGNED_STACK=0 -DPREFIX)
        endif()
    endif()
    if(WIN32)
        set(SUFFIX obj)
    else()
        set(SUFFIX o)
    endif()
    if (HIGH_BIT_DEPTH)
        set(FLAGS ${FLAGS} -DHIGH_BIT_DEPTH=1 -DBIT_DEPTH=10)
    else()
        set(FLAGS ${FLAGS} -DHIGH_BIT_DEPTH=0 -DBIT_DEPTH=8)
    endif()
    foreach(ASM ${MSVC_ASMS})
        set(YASM_SRC ${CMAKE_CURRENT_SOURCE_DIR}/common/x86/${ASM})
        set(YASM_SRCS ${YASM_SRCS} ${YASM_SRC})
        set(YASM_OBJS ${YASM_OBJS} ${ASM}.${SUFFIX})
        add_custom_command(
            OUTPUT ${ASM}.${SUFFIX}
            COMMAND ${YASM_EXECUTABLE} ARGS ${FLAGS} ${YASM_SRC} -o ${ASM}.${SUFFIX}
            DEPENDS ${YASM_SRC})
    endforeach()
endif()

add_library(x265-static STATIC $<TARGET_OBJECTS:encoder> $<TARGET_OBJECTS:common> ${YASM_OBJS} ${YASM_SRCS})
if(NOT MSVC)
    set_target_properties(x265-static PROPERTIES OUTPUT_NAME x265)
endif()
install(TARGETS x265-static
        RUNTIME DESTINATION bin
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib)
install(FILES x265.h "${PROJECT_BINARY_DIR}/x265_config.h" DESTINATION include)

option(ENABLE_SHARED "Build shared library" ON)
if(ENABLE_SHARED)
    add_library(x265-shared SHARED dllmain.cpp "${PROJECT_BINARY_DIR}/x265.def" ${YASM_OBJS}
                                   $<TARGET_OBJECTS:encoder> $<TARGET_OBJECTS:common>)
    target_link_libraries(x265-shared ${PLATFORM_LIBS})
    set_target_properties(x265-shared PROPERTIES OUTPUT_NAME x265)
    if(UNIX AND NOT APPLE)
        set_target_properties(x265-shared PROPERTIES LINK_FLAGS "-Wl,-Bsymbolic")
    endif()
    if(X265_LATEST_TAG)
        # shared library is not installed if a tag is not found
        set_target_properties(x265-shared PROPERTIES VERSION ${X265_LATEST_TAG} SOVERSION ${X265_BUILD})
        install(TARGETS x265-shared
                RUNTIME DESTINATION bin
                LIBRARY DESTINATION lib
                ARCHIVE DESTINATION lib)

        # convert lists of link libraries into -lstdc++ -lm etc..
        foreach(LIB ${CMAKE_CXX_IMPLICIT_LINK_LIBRARIES} ${PLATFORM_LIBS})
            set(PRIVATE_LIBS "${PRIVATE_LIBS} -l${LIB}")
        endforeach()
        # Produce a pkg-config file for linking against the shared lib
        configure_file("x265.pc.in" "x265.pc" @ONLY)
        install(FILES       "${CMAKE_CURRENT_BINARY_DIR}/x265.pc"
                DESTINATION "${CMAKE_INSTALL_PREFIX}/lib/pkgconfig")
    endif()
endif()

if(NOT WIN32)
    configure_file("${CMAKE_CURRENT_SOURCE_DIR}/cmake/cmake_uninstall.cmake.in"
                   "${CMAKE_CURRENT_BINARY_DIR}/cmake/cmake_uninstall.cmake"
                   IMMEDIATE @ONLY)
    add_custom_target(uninstall
                      "${CMAKE_COMMAND}" -P "${CMAKE_CURRENT_BINARY_DIR}/cmake/cmake_uninstall.cmake")
endif()

# Main CLI application
option(ENABLE_CLI "Build standalone CLI application" ON)
if(ENABLE_CLI)
    file(GLOB InputFiles input/*.cpp input/*.h)
    file(GLOB OutputFiles output/*.cpp output/*.h)
    source_group(input FILES ${InputFiles})
    source_group(output FILES ${OutputFiles})

    check_include_files(getopt.h HAVE_GETOPT_H)
    if(NOT HAVE_GETOPT_H)
        if(MSVC)
            set_source_files_properties(compat/getopt/getopt.c PROPERTIES COMPILE_FLAGS "/wd4100 /wd4131 -DHAVE_STRING_H=1")
        endif(MSVC)
        include_directories(compat/getopt)
        set(GETOPT compat/getopt/getopt.c compat/getopt/getopt.h)
    endif(NOT HAVE_GETOPT_H)

    if(XCODE)
        add_executable(cli ../COPYING ${InputFiles} ${OutputFiles} ${GETOPT} x265.cpp x265.h
                           $<TARGET_OBJECTS:encoder> $<TARGET_OBJECTS:common> ${YASM_OBJS} ${YASM_SRCS})
    else()
        add_executable(cli ../COPYING ${InputFiles} ${OutputFiles} ${GETOPT} x265.cpp x265.h)
        target_link_libraries(cli x265-static ${PLATFORM_LIBS})
    endif()
    set_target_properties(cli PROPERTIES OUTPUT_NAME x265)

    install(TARGETS cli DESTINATION bin)
endif(ENABLE_CLI)

if(ENABLE_ASSEMBLY AND NOT XCODE)
    option(ENABLE_TESTS "Enable Unit Tests" OFF)
    if(ENABLE_TESTS)
        add_subdirectory(test)
    endif()
endif()
