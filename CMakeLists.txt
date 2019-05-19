cmake_minimum_required(VERSION 3.2)

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
  message(STATUS "Setting default build type: ${CMAKE_BUILD_TYPE}")
endif()
string(TOLOWER ${CMAKE_BUILD_TYPE} CMAKE_BUILD_TYPE_LOWER)

enable_language(C)
enable_language(CXX)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
find_package(Threads)

function(append_if condition value)
  if (${condition})
    foreach(variable ${ARGN})
      set(${variable} "${${variable}} ${value}" PARENT_SCOPE)
    endforeach(variable)
  endif()
endfunction()


cmake_policy(SET CMP0048 NEW)

set(VERSION_MAJOR "1")
set(VERSION_MINOR "0")
set(VERSION_PATCH "0")
# set(VERSION_BUILD_TYPE "${CMAKE_BUILD_TYPE}")
set(VERSION_IS_DIRTY "")
string(TIMESTAMP timestamp "%y%m%d.%H%M%S" UTC)
set(VERSION_BUILD_TIME "${timestamp}")
# $Format:Packaged from commit %H%nset(COMMIT %h)%nset(REFS "%d")$

set_property(GLOBAL PROPERTY USE_FOLDERS ON)
set(CMAKE_CONFIGURATION_TYPES Debug RelWithDebInfo Release CACHE TYPE INTERNAL)
set(CMAKE_SKIP_INSTALL_RULES OFF)
set(CMAKE_SKIP_PACKAGE_ALL_DEPENDENCY ON)
set(CMAKE_SUPPRESS_REGENERATION ON)
set(BUILD_TESTS ON CACHE STRING "Spend time making tests? Defaults to ON")
set(STATIC ON CACHE BOOL "Link libraries statically")
set(PORTABLE ON CACHE STRING "Defaults to ON")

if(BUILD_TESTS)
  enable_testing()
endif()

# copy CTestCustom.cmake to build dir to disable long running tests in 'make test'
configure_file(${CMAKE_SOURCE_DIR}/CTestCustom.cmake ${CMAKE_BINARY_DIR})

project(Adon)

include_directories(include src external "${CMAKE_BINARY_DIR}/version")
if(APPLE)
  include_directories(SYSTEM /usr/include/malloc)
  enable_language(ASM)
endif()

if(MSVC)
  include_directories(${CMAKE_CURRENT_SOURCE_DIR}/src/Platform/Windows)
elseif(APPLE)
  include_directories(${CMAKE_CURRENT_SOURCE_DIR}/src/Platform/OSX)
  include_directories(${CMAKE_CURRENT_SOURCE_DIR}/src/Platform/Posix)
else()
  include_directories(${CMAKE_CURRENT_SOURCE_DIR}/src/Platform/Linux)
  include_directories(${CMAKE_CURRENT_SOURCE_DIR}/src/Platform/Posix)
endif()

set(STATIC ${MSVC} CACHE BOOL "Link libraries statically")

if(MSVC)
  add_definitions("/bigobj /MP /W3 /GS- /D_CRT_SECURE_NO_WARNINGS /wd4996 /wd4345 /D_WIN32_WINNT=0x0600 /DWIN32_LEAN_AND_MEAN /DGTEST_HAS_TR1_TUPLE=0 /D_VARIADIC_MAX=8 /D__SSE4_1__")
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /STACK:10485760")
  if(STATIC)
    foreach(VAR CMAKE_C_FLAGS_DEBUG CMAKE_CXX_FLAGS_DEBUG CMAKE_C_FLAGS_RELWITHDEBINFO CMAKE_CXX_FLAGS_RELWITHDEBINFO CMAKE_C_FLAGS_RELEASE CMAKE_CXX_FLAGS_RELEASE)
      string(REPLACE "/MD" "/MT" ${VAR} "${${VAR}}")
    endforeach()
  endif()
  include_directories(SYSTEM src/platform/msc)
else()
  if (${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
    # This option has no effect in glibc version less than 2.20. 
    # Since glibc 2.20 _BSD_SOURCE is deprecated, this macro is recomended instead
    add_definitions("-D_DEFAULT_SOURCE -D_GNU_SOURCE")
  endif()
  set(ARCH native CACHE STRING "CPU to build for: -march value or default")
  if("${ARCH}" STREQUAL "default")
    set(ARCH_FLAG "")
  else()
    set(ARCH_FLAG "-march=${ARCH}")
  endif()
  set(WARNINGS "-Wall -Wextra -Wpointer-arith -Wundef -Wvla -Wwrite-strings -Wno-error=extra -Wno-error=unused-function -Wno-error=deprecated-declarations -Wno-error=sign-compare -Wno-error=strict-aliasing -Wno-error=type-limits -Wno-unused-parameter -Wno-error=unused-variable -Wno-error=undef -Wno-error=uninitialized -Wno-error=unused-result")
  if(CMAKE_C_COMPILER_ID STREQUAL "Clang")
    set(WARNINGS "${WARNINGS} -Wno-error=mismatched-tags -Wno-unused-lambda-capture -Wno-error=null-conversion -Wno-overloaded-shift-op-parentheses -Wno-error=shift-count-overflow -Wno-error=tautological-constant-out-of-range-compare -Wno-error=unused-private-field -Wno-error=unneeded-internal-declaration -Wno-error=unused-function -Wno-error=missing-braces")
	endif()
  if(MINGW)
    set(WARNINGS "${WARNINGS} -Wno-error=unused-value")
    set(MINGW_FLAG "-DWIN32_LEAN_AND_MEAN")
    include_directories(SYSTEM src/platform/mingw)
  else()
    set(MINGW_FLAG "")
  endif()
  if(CMAKE_C_COMPILER_ID STREQUAL "GNU" AND NOT (CMAKE_C_COMPILER_VERSION VERSION_LESS 5.1))
    set(WARNINGS "${WARNINGS} -Wno-error=odr")
  endif()
  set(C_WARNINGS "-Waggregate-return -Wnested-externs -Wold-style-definition -Wstrict-prototypes")
  set(CXX_WARNINGS "-Wno-reorder -Wno-missing-field-initializers")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=c11 ${MINGW_FLAG} ${WARNINGS} ${C_WARNINGS} ${ARCH_FLAG} -maes")
  if(NOT APPLE)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pthread")
  endif()
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 ${MINGW_FLAG} ${WARNINGS} ${CXX_WARNINGS} ${ARCH_FLAG} -maes")
  if(APPLE)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DGTEST_HAS_TR1_TUPLE=0")
  endif()
  if(CMAKE_C_COMPILER_ID STREQUAL "GNU" AND NOT (CMAKE_C_COMPILER_VERSION VERSION_LESS 4.8))
    set(DEBUG_FLAGS "-g3 -Og")
  else()
    set(DEBUG_FLAGS "-g3 -O0")
  endif()
  set(RELEASE_FLAGS "-Ofast -DNDEBUG -Wno-unused-variable")
  if(NOT APPLE)
    # There is a clang bug that does not allow to compile code that uses AES-NI intrinsics if -flto is enabled
    if (CMAKE_C_COMPILER_ID STREQUAL "GNU" AND CMAKE_SYSTEM_NAME STREQUAL "Linux" 
        AND CMAKE_BUILD_TYPE STREQUAL "Release" AND ((CMAKE_C_COMPILER_VERSION VERSION_GREATER 4.9) OR (CMAKE_C_COMPILER_VERSION VERSION_EQUAL 4.9)))
      # On linux, to build in lto mode, check that ld.gold linker is used: 'update-alternatives --install /usr/bin/ld ld /usr/bin/ld.gold HIGHEST_PRIORITY'
      set(CMAKE_AR gcc-ar)
      set(CMAKE_RANLIB gcc-ranlib)
    endif()
  set(RELEASE_FLAGS "${RELEASE_FLAGS} -flto")
  endif()

    # With GCC 6.1.1 the compiled binary malfunctions due to aliasing. Until that
  # is fixed in the code (Issue #847), force compiler to be conservative.
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fno-strict-aliasing")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-strict-aliasing")

  if(NOT DEFINED USE_LTO_DEFAULT)
    set(USE_LTO_DEFAULT false)
  endif()
  set(USE_LTO ${USE_LTO_DEFAULT} CACHE BOOL "Use Link-Time Optimization (Release mode only)")

  if(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    # There is a clang bug that does not allow to compile code that uses AES-NI intrinsics if -flto is enabled, so explicitly disable
    set(USE_LTO false)
  endif()

  if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU" OR
      CMAKE_CXX_COMPILER_ID STREQUAL "Clang" OR
      CMAKE_CXX_COMPILER_ID STREQUAL "AppleClang")
    # Unify visibility to meet llvm's default.
    include(CheckCXXCompilerFlag)

    check_cxx_compiler_flag("-fvisibility-inlines-hidden" SUPPORTS_FVISIBILITY_INLINES_HIDDEN_FLAG)
    append_if(SUPPORTS_FVISIBILITY_INLINES_HIDDEN_FLAG "-fvisibility-inlines-hidden" CMAKE_CXX_FLAGS)

    # Enable standard warnings.
    # set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra")

    # Enable additional warnings that are not included in -Wall and -Wextra.
    # set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wcast-align")
    # set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wcast-qual")
    # set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wswitch-default")
    # set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wuninitialized")
    # set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wold-style-cast")

    # Disable warnings that produce more headaches than use.
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-unused-parameter")
  endif()

  if(USE_LTO)
    set(RELEASE_FLAGS "${RELEASE_FLAGS} -flto")
    if(STATIC)
      set(RELEASE_FLAGS "${RELEASE_FLAGS} -ffat-lto-objects")
    endif()
    # Since gcc 4.9 the LTO format is non-standard (slim), so we need the gcc-specific ar and ranlib binaries
    if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU" AND NOT (CMAKE_CXX_COMPILER_VERSION VERSION_LESS 4.9.0) AND NOT OPENBSD AND NOT DRAGONFLY)
      # When invoking cmake on distributions on which gcc's binaries are prefixed
      # with an arch-specific triplet, the user must specify -DCHOST=<prefix>
      if (DEFINED CHOST)
        set(CMAKE_AR "${CHOST}-gcc-ar")
        set(CMAKE_RANLIB "${CHOST}-gcc-ranlib")
      else()
        set(CMAKE_AR "gcc-ar")
        set(CMAKE_RANLIB "gcc-ranlib")
      endif()
    endif()
  endif()

  set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} ${DEBUG_FLAGS}")
  set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} ${DEBUG_FLAGS}")
  set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} ${RELEASE_FLAGS}")
  set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} ${RELEASE_FLAGS}")
  if(STATIC AND NOT APPLE)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -static-libgcc -static-libstdc++")
  endif()
endif()

if(APPLE)
  set(STATIC ON)
endif()

if(STATIC)
  set(Boost_USE_STATIC_LIBS ON)
  set(Boost_USE_STATIC_RUNTIME ON)
endif()

find_package(Boost 1.55 REQUIRED COMPONENTS system filesystem thread date_time chrono regex serialization program_options)
include_directories(SYSTEM ${Boost_INCLUDE_DIRS})

if(MINGW)
  set(Boost_LIBRARIES "${Boost_LIBRARIES};ws2_32;mswsock")
elseif(APPLE)
  set(Boost_LIBRARIES "${Boost_LIBRARIES}")
elseif(NOT MSVC)
  set(Boost_LIBRARIES "${Boost_LIBRARIES};rt")
endif()

set(COMMIT_ID_IN_VERSION ON CACHE BOOL "Include commit ID in version")
file(MAKE_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/version")
if (NOT COMMIT_ID_IN_VERSION)
  set(VERSION "${VERSION}-unknown")
  set(COMMIT_ID "unknown")
  set(VERSION_DIRTY "")
  message(STATUS "Version : ${VERSION}")
  message(STATUS "Revision : ${COMMIT_ID}")
  configure_file("${CMAKE_CURRENT_SOURCE_DIR}/src/version.h.in" "${CMAKE_CURRENT_BINARY_DIR}/version/version.h")
  add_custom_target(version ALL)
elseif(DEFINED COMMIT_ID_IN_VERSION)
  find_package(Git QUIET)
  if(Git_FOUND OR GIT_FOUND)
    message(STATUS "Found Git: ${GIT_EXECUTABLE}")
    execute_process(
        COMMAND git rev-parse --abbrev-ref HEAD  # Get the current branch
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        OUTPUT_VARIABLE GIT_BRANCH
        OUTPUT_STRIP_TRAILING_WHITESPACE
        )

    execute_process(
        COMMAND git log -1 --format=%h # Get latest commit hash
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        OUTPUT_VARIABLE COMMIT_ID
        OUTPUT_STRIP_TRAILING_WHITESPACE
        )

    execute_process(
        COMMAND git describe --tags --dirty=-Dirty # Get latest commit tags
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        OUTPUT_VARIABLE TAGS
        OUTPUT_STRIP_TRAILING_WHITESPACE
        )
        string(REGEX REPLACE "^v([0-9]+)\\..*" "\\1" VERSION_MAJOR "${TAGS}")
        string(REGEX REPLACE "^v[0-9]+\\.([0-9]+).*" "\\1" VERSION_MINOR "${TAGS}")
        string(REGEX REPLACE "^v[0-9]+\\.[0-9]+\\.([0-9]+).*" "\\1" VERSION_PATCH "${TAGS}")
        string(REGEX REPLACE "^v[0-9]+\\.[0-9]+\\.[0-9]+(.*)" "\\1" VERSION_SHA1 "${TAGS}")
        if(${TAGS} MATCHES "Dirty$")
          set(VERSION_DIRTY "Dirty")
        endif()
        set(VERSION_SHORT "${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}.${COMMIT_ID} ${VERSION_DIRTY}")

  else()
      message(STATUS "Git not found !")
      set(GIT_BRANCH "unknown")
      set(COMMIT_ID "unknown")
  endif()
  set(VERSION "${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}")
  set(VERSION_BUILD_TYPE "${CMAKE_BUILD_TYPE}")
  configure_file("${CMAKE_CURRENT_SOURCE_DIR}/src/version.h.in" "${CMAKE_CURRENT_BINARY_DIR}/version/version.h")
  add_custom_target(version ALL)
endif()

add_subdirectory(external)
add_subdirectory(src)
if(BUILD_TESTS)
  add_subdirectory(tests)
endif()
 
string(TIMESTAMP timestamp "%y%m%d.%H%M%S" UTC)

message(STATUS "------------------------- ADON BUILD DETALS ------------------------")
message(STATUS "version     : ${VERSION_SHORT}")
message(STATUS "Build       : ${VERSION_BUILD_TYPE}")
message(STATUS "Compiler    : ${CMAKE_C_COMPILER_ID} version ${CMAKE_CXX_COMPILER_VERSION}")
message(STATUS "Branch      : ${GIT_BRANCH}")
message(STATUS "Version     : ${VERSION} [${VERSION_BUILD_TYPE}.${COMMIT_ID} ${VERSION_DIRTY} ${VERSION_BUILD_TIME}]")
message(STATUS "Boost       : ${Boost_LIB_VERSION}")
message(STATUS "Libs Static : ${STATIC}")
message(STATUS "Portable    : ${PORTABLE}" )
message(STATUS "Tests       : ${BUILD_TESTS}")
message(STATUS "Time        : ${timestamp}")
message(STATUS "---------------------------------------------------------------------")