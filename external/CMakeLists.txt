set(UPNPC_BUILD_STATIC ON CACHE BOOL "Build static library")
set(UPNPC_BUILD_SHARED OFF CACHE BOOL "Build shared library")
set(UPNPC_BUILD_TESTS OFF CACHE BOOL "Build test executables")

if (STATIC AND MSVC)
  foreach(VAR CMAKE_C_FLAGS_RELEASE)
    string(REPLACE "/MD" "/MT" ${VAR} "${${VAR}}")
  endforeach()
endif()

add_subdirectory(miniupnpc)
add_subdirectory(argon2)

set_property(TARGET upnpc-static argon2 PROPERTY FOLDER "external")

if(MSVC)
  set_property(TARGET upnpc-static APPEND_STRING PROPERTY COMPILE_FLAGS " -wd4244 -wd4267")
elseif(FREEBSD)
  set_property(TARGET upnpc-static APPEND_STRING PROPERTY COMPILE_FLAGS " -D_XOPEN_SOURCE=600 -Wno-undef -Wno-unused-result -Wno-unused-value")
else()
  set_property(TARGET upnpc-static APPEND_STRING PROPERTY COMPILE_FLAGS " -Wno-undef -Wno-unused-result -Wno-unused-value")
endif()
