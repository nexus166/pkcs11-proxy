cmake_minimum_required (VERSION 2.4)

# Gemini: Resolve CMP0042 (macOS-only).
if (APPLE)
  set(CMAKE_MACOSX_RPATH 1)
endif()

if(COMMAND cmake_policy)
      cmake_policy(SET CMP0003 NEW)
endif(COMMAND cmake_policy)

project (pkcs11 C)

# Gemini: syscall-reporter.c and syscall-names.h apply only to Linux. Build only in debug mode.
set(PKCS11_PROXY_SRCS gck-rpc-module.c gck-rpc-message.c gck-rpc-util.c egg-buffer.c gck-rpc-tls-psk.c)
set(PKCS11_DAEMON_SRCS egg-buffer.c gck-rpc-daemon-standalone.c gck-rpc-dispatch.c gck-rpc-message.c gck-rpc-util.c gck-rpc-tls-psk.c)
macro (CHECK_IF_DEBUG)
  if (CMAKE_BUILD_TYPE MATCHES Debug)
    if (NOT APPLE)
      set(PKCS11_DAEMON_SRCS ${PKCS11_DAEMON_SRCS} syscall-reporter.c syscall-names.h)
    endif()
  endif()
endmacro()

add_definitions(-Wall)
add_library(pkcs11-proxy SHARED ${PKCS11_PROXY_SRCS})

# Disable console when building Win32 binary in release mode
if (WIN32)
  if("${CMAKE_BUILD_TYPE}" MATCHES "^Rel.*")
    set(GUI_TYPE WIN32)
  endif()
endif()

add_executable (pkcs11-daemon ${GUI_TYPE} ${PKCS11_DAEMON_SRCS})

# Gemini: Linking OpenSSL on macOS is a bit fragile. We can't link Apple's version of
# OpenSSL (https://developer.apple.com/forums/thread/124782?answerId=390243022#390243022).
# Instead, we need to install the latest version via brew, confirm the package is found,
# and then add an include directory (here) and link directory (below).
if (APPLE)
  find_package(OpenSSL REQUIRED)
  include_directories (/usr/local/opt/openssl/include)
endif()

set_target_properties(pkcs11-proxy PROPERTIES VERSION 0.1 SOVERSION 0)

if (WIN32)
  include_directories("ext/include")

  add_library(dl STATIC IMPORTED)
  set_property(TARGET dl PROPERTY
               IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/ext/lib/libdl.a)
  target_link_libraries (pkcs11-daemon ws2_32)
  target_link_libraries (pkcs11-proxy ws2_32)
endif (WIN32)

# Gemini: Link directory added.
if (APPLE)
  target_link_libraries (pkcs11-proxy -L/usr/local/opt/openssl/lib ssl crypto pthread)
  target_link_libraries (pkcs11-daemon dl -L/usr/local/opt/openssl/lib ssl crypto pthread)
else()
  target_link_libraries (pkcs11-proxy ssl crypto pthread)
  target_link_libraries (pkcs11-daemon dl ssl crypto pthread seccomp)
endif()

install_targets (/lib pkcs11-proxy)
install_targets (/bin pkcs11-daemon)

# Gemini: Don't perform command on macOS.
if (NOT APPLE)
  add_custom_command(
     OUTPUT syscall-names.h
     COMMAND ${CMAKE_SOURCE_DIR}/mksyscalls.sh
     WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
endif()
