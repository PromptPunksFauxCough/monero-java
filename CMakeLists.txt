cmake_minimum_required(VERSION 3.24)
project(monero-java-jni)

option(BUILD_STATIC "Build a monolithic JNI wrapper that statically links monero-cpp and all its dependencies." OFF)

include(FetchContent)
include(ExternalProject)

if(WIN32)
  add_definitions("-D_GLIBCXX_USE_NANOSLEEP=1")
  add_definitions("-DWIN32_LEAN_AND_MEAN")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wa,-mbig-obj -O2 -fPIC -std=c++14 -F/Library/Frameworks -pthread -lcrypto -lcrypt32 -lbcrypt")
else()
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC -std=c++14 -F/Library/Frameworks -pthread")
endif()

if(NOT DEFINED ENV{JAVA_HOME} OR "$ENV{JAVA_HOME}" STREQUAL "")
  message(FATAL_ERROR "JAVA_HOME variable not set, e.g.: export JAVA_HOME=/path/to/jdk")
endif()

##########################
# Monero-cpp fetch
##########################
set(MONERO_CPP_DIR "${CMAKE_SOURCE_DIR}/external/monero-cpp")
set(MONERO_CPP_COMMIT "4e0268655f09e84653210506a6e320b51527434e")

FetchContent_Declare(
  monero_cpp_external
  GIT_REPOSITORY https://github.com/woodser/monero-cpp
  GIT_TAG ${MONERO_CPP_COMMIT}
  SOURCE_DIR ${MONERO_CPP_DIR}
)
FetchContent_Populate(monero_cpp_external)

##########################
# Monero-project fetch
##########################
set(MONERO_PROJECT_DIR "${MONERO_CPP_DIR}/external/monero-project")
set(MONERO_PROJECT_BUILD_DIR "${CMAKE_BINARY_DIR}/external/monero-build")
set(MONERO_PROJECT_COMMIT "ec870e50706a29768a65f597155ed5c7ad7e6326")

if(NOT EXISTS "${MONERO_PROJECT_DIR}/CMakeLists.txt")
  message(STATUS "Fetching Monero into ${MONERO_PROJECT_DIR}...")
  FetchContent_Declare(
    monero
    GIT_REPOSITORY https://github.com/monero-project/monero.git
    GIT_TAG ${MONERO_PROJECT_COMMIT}
    SOURCE_DIR ${MONERO_PROJECT_DIR}
  )
  set(FETCHCONTENT_UPDATES_DISCONNECTED ON)
  FetchContent_Populate(monero)
endif()

ExternalProject_Add(monero_ext
  SOURCE_DIR ${MONERO_PROJECT_DIR}
  BINARY_DIR ${MONERO_PROJECT_DIR}/build/release
  UPDATE_DISCONNECTED 1
  CMAKE_ARGS
    # General Build Options
    -DBUILD_SHARED_LIBS=OFF
    -DBUILD_GUI=OFF
    -DCMAKE_INSTALL_PREFIX=${MONERO_PROJECT_DIR}/build/install

    # Dependency Root Directories
    -DBOOST_ROOT=${BOOST_ROOT}
    -DBOOST_LIBRARY_DIR=${BOOST_LIBRARY_DIR}
    -DBOOST_LIBRARIES=${BOOST_LIBRARIES}
    -DBoost_ARCHITECTURE=${Boost_ARCHITECTURE}
    -DBoost_CHRONO_LIBRARY_RELEASE=${Boost_CHRONO_LIBRARY_RELEASE}
    -DBoost_DATE_TIME_LIBRARY_RELEASE=${Boost_DATE_TIME_LIBRARY_RELEASE}
    -DBoost_FILESYSTEM_LIBRARY_RELEASE=${Boost_FILESYSTEM_LIBRARY_RELEASE}
    -DBoost_PROGRAM_OPTIONS_LIBRARY_RELEASE=${Boost_PROGRAM_OPTIONS_LIBRARY_RELEASE}
    -DBoost_REGEX_LIBRARY_RELEASE=${Boost_REGEX_LIBRARY_RELEASE}
    -DBoost_SERIALIZATION_LIBRARY_RELEASE=${Boost_SERIALIZATION_LIBRARY_RELEASE}
    -DBoost_SYSTEM_LIBRARY_RELEASE=${Boost_SYSTEM_LIBRARY_RELEASE}
    -DBoost_THREAD_LIBRARY_RELEASE=${Boost_THREAD_LIBRARY_RELEASE}
    -DBoost_LOCALE_LIBRARY_RELEASE=${Boost_LOCALE_LIBRARY_RELEASE}
    -DOPENSSL_ROOT_DIR=${OPENSSL_ROOT_DIR}
    -DLIBSODIUM_ROOT_DIR=${LIBSODIUM_ROOT_DIR}
    -DHIDAPI_LIBRARY_DIR=${HIDAPI_LIBRARY_DIR}
    -DHIDAPI_LIBRARY=${HIDAPI_LIBRARY}
    -DHIDAPI_INCLUDE_DIR=${HIDAPI_INCLUDE_DIR}
    -DReadline_ROOT_DIR=${Readline_ROOT_DIR}
    -DReadline_INCLUDE_DIR=${Readline_INCLUDE_DIR}
    -DReadline_LIBRARY=${Readline_LIBRARY}
    -DZeroMQ_ROOT_DIR=${ZeroMQ_ROOT_DIR}
    -DZeroMQ_INCLUDE_DIR=${ZeroMQ_INCLUDE_DIR}
    -DZeroMQ_LIBRARY=${ZeroMQ_LIBRARY}
    -DLIBUSB_ROOT_DIR=${LIBUSB_ROOT_DIR}
    -DUNBOUND_LIBRARY_DIR=${UNBOUND_LIBRARY_DIR}
    -DUNBOUND_LIBRARY=${UNBOUND_LIBRARY}
    -DUNBOUND_LIBRARIES=${UNBOUND_LIBRARIES}
    -DUNBOUND_INCLUDE_DIR=${UNBOUND_INCLUDE_DIR}
    -DLIBUNWIND_LIBRARY=${LIBUNWIND_LIBRARY}
    -DLIBUNWIND_LIBRARIES=${LIBUNWIND_LIBRARIES}
    -DLIBUNWIND_INCLUDE_DIR=${LIBUNWIND_INCLUDE_DIR}
    -DBacktrace_LIBRARY=${Backtrace_LIBRARY}
    -DBacktrace_INCLUDE_DIR=${Backtrace_INCLUDE_DIR}
    -DProtobuf_PROTOC_EXECUTABLE=${Protobuf_PROTOC_EXECUTABLE}
    -DProtobuf_LIBRARY=${Protobuf_LIBRARY}
    -DProtobuf_INCLUDE_DIR=${Protobuf_INCLUDE_DIR}

  BUILD_COMMAND ${CMAKE_COMMAND} --build <BINARY_DIR> --target wallet_api && ${CMAKE_COMMAND} --build <BINARY_DIR> --
  INSTALL_COMMAND ""
  BUILD_BYPRODUCTS ${MONERO_PROJECT_DIR}/build/release/lib/libwallet_api.a
)

# Register Monero libwallet_api statically from external build
add_library(wallet_api STATIC IMPORTED GLOBAL)
set_target_properties(wallet_api PROPERTIES
  IMPORTED_LOCATION "${MONERO_PROJECT_DIR}/build/release/lib/libwallet_api.a"
  INTERFACE_INCLUDE_DIRECTORIES "${MONERO_PROJECT_DIR}/src/wallet/api"
)
add_dependencies(wallet_api monero_ext)

##########################
# monero-cpp build
##########################
ExternalProject_Add(monero_cpp_build
  SOURCE_DIR ${MONERO_CPP_DIR}
  BINARY_DIR ${MONERO_CPP_DIR}/build
  CMAKE_ARGS
    # General Build Options
    -DCMAKE_BUILD_TYPE=Release
    -DBUILD_STATIC=${BUILD_STATIC}
    -DCMAKE_INSTALL_PREFIX=${MONERO_CPP_DIR}/install

    # Dependency Root Directories
    -DBOOST_ROOT=${BOOST_ROOT}
    -DBOOST_LIBRARY_DIR=${BOOST_LIBRARY_DIR}
    -DBOOST_LIBRARIES=${BOOST_LIBRARIES}
    -DBoost_ARCHITECTURE=${Boost_ARCHITECTURE}
    -DBoost_CHRONO_LIBRARY_RELEASE=${Boost_CHRONO_LIBRARY_RELEASE}
    -DBoost_DATE_TIME_LIBRARY_RELEASE=${Boost_DATE_TIME_LIBRARY_RELEASE}
    -DBoost_FILESYSTEM_LIBRARY_RELEASE=${Boost_FILESYSTEM_LIBRARY_RELEASE}
    -DBoost_PROGRAM_OPTIONS_LIBRARY_RELEASE=${Boost_PROGRAM_OPTIONS_LIBRARY_RELEASE}
    -DBoost_REGEX_LIBRARY_RELEASE=${Boost_REGEX_LIBRARY_RELEASE}
    -DBoost_SERIALIZATION_LIBRARY_RELEASE=${Boost_SERIALIZATION_LIBRARY_RELEASE}
    -DBoost_SYSTEM_LIBRARY_RELEASE=${Boost_SYSTEM_LIBRARY_RELEASE}
    -DBoost_THREAD_LIBRARY_RELEASE=${Boost_THREAD_LIBRARY_RELEASE}
    -DBoost_LOCALE_LIBRARY_RELEASE=${Boost_LOCALE_LIBRARY_RELEASE}
    -DOPENSSL_ROOT_DIR=${OPENSSL_ROOT_DIR}
    -DLIBSODIUM_ROOT_DIR=${LIBSODIUM_ROOT_DIR}
    -DHIDAPI_LIBRARY_DIR=${HIDAPI_LIBRARY_DIR}
    -DHIDAPI_LIBRARY=${HIDAPI_LIBRARY}
    -DHIDAPI_INCLUDE_DIR=${HIDAPI_INCLUDE_DIR}
    -DReadline_ROOT_DIR=${Readline_ROOT_DIR}
    -DReadline_INCLUDE_DIR=${Readline_INCLUDE_DIR}
    -DReadline_LIBRARY=${Readline_LIBRARY}
    -DZeroMQ_ROOT_DIR=${ZeroMQ_ROOT_DIR}
    -DZeroMQ_INCLUDE_DIR=${ZeroMQ_INCLUDE_DIR}
    -DZeroMQ_LIBRARY=${ZeroMQ_LIBRARY}
    -DLIBUSB_ROOT_DIR=${LIBUSB_ROOT_DIR}
    -DUNBOUND_LIBRARY_DIR=${UNBOUND_LIBRARY_DIR}
    -DUNBOUND_LIBRARY=${UNBOUND_LIBRARY}
    -DUNBOUND_LIBRARIES=${UNBOUND_LIBRARIES}
    -DUNBOUND_INCLUDE_DIR=${UNBOUND_INCLUDE_DIR}
    -DLIBUNWIND_LIBRARY=${LIBUNWIND_LIBRARY}
    -DLIBUNWIND_LIBRARIES=${LIBUNWIND_LIBRARIES}
    -DLIBUNWIND_INCLUDE_DIR=${LIBUNWIND_INCLUDE_DIR}
    -DBacktrace_LIBRARY=${Backtrace_LIBRARY}
    -DBacktrace_INCLUDE_DIR=${Backtrace_INCLUDE_DIR}
    -DProtobuf_ROOT_DIR=${Protobuf_ROOT_DIR}
    -DProtobuf_PROTOC_EXECUTABLE=${Protobuf_PROTOC_EXECUTABLE}
    -DProtobuf_LIBRARY=${Protobuf_LIBRARY}
    -DProtobuf_INCLUDE_DIR=${Protobuf_INCLUDE_DIR}
    -DMONERO_PROJECT_DIR=${MONERO_PROJECT_DIR}
    -DMONERO_PROJECT_BUILD=${MONERO_PROJECT_DIR}/build/release

  UPDATE_COMMAND ""
#  PATCH_COMMAND ""
  PATCH_COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/patches/monero-cpp.patch ${MONERO_CPP_DIR} && patch -p1 -d ${MONERO_CPP_DIR} -i ${MONERO_CPP_DIR}/monero-cpp.patch
  CONFIGURE_HANDLED_BY_BUILD ON
)

function(register_monero_cpp_lib name relpath)
  add_library(${name} STATIC IMPORTED)
  set_target_properties(${name} PROPERTIES
    IMPORTED_LOCATION ${MONERO_PROJECT_BUILD_DIR}/build/${relpath}
  )
  add_dependencies(${name} monero_ext)
endfunction()

# Register Monero static libs (as needed)
register_monero_cpp_lib(libwallet                lib/libwallet.a)

add_library(monero-cpp SHARED IMPORTED GLOBAL)

if(APPLE)
  set_target_properties(monero-cpp PROPERTIES IMPORTED_LOCATION "${MONERO_CPP_DIR}/build/libmonero-cpp.dylib")
elseif(WIN32)
  set_target_properties(monero-cpp PROPERTIES
    IMPORTED_LOCATION "${MONERO_CPP_DIR}/build/libmonero-cpp.dll"
    IMPORTED_IMPLIB "${MONERO_CPP_DIR}/build/libmonero-cpp.dll.a"
  )
else()
  set_target_properties(monero-cpp PROPERTIES IMPORTED_LOCATION "${MONERO_CPP_DIR}/build/libmonero-cpp.so")
endif()

add_dependencies(monero-cpp monero_cpp_build)

##########################
# Boost and OpenSSL
##########################
set(Boost_NO_BOOST_CMAKE 1)
set(Boost_USE_MULTITHREADED ON)
find_package(Boost 1.58 QUIET REQUIRED COMPONENTS chrono date_time filesystem program_options regex serialization wserialization system thread)
message(STATUS "Using Boost include dir at ${Boost_INCLUDE_DIR}")

if(APPLE AND NOT IOS)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fvisibility=default")
  if(NOT OPENSSL_ROOT_DIR)
    execute_process(COMMAND brew --prefix openssl
      OUTPUT_VARIABLE OPENSSL_ROOT_DIR
      OUTPUT_STRIP_TRAILING_WHITESPACE)
    message(STATUS "Using OpenSSL found at ${OPENSSL_ROOT_DIR}")
  endif()
endif()

find_package(OpenSSL REQUIRED)
message(STATUS "Using OpenSSL include dir at ${OPENSSL_INCLUDE_DIR}")

if(STATIC AND NOT IOS AND UNIX)
  set(OPENSSL_LIBRARIES "${OPENSSL_LIBRARIES};${CMAKE_DL_LIBS};${CMAKE_THREAD_LIBS_INIT}")
endif()

if(WIN32)
  list(APPEND OPENSSL_LIBRARIES ws2_32 crypt32 bcrypt)
endif()

##########################
# Platform extras
##########################
set(EXTRA_LIBRARIES "")
if(APPLE)
  find_library(COREFOUNDATION CoreFoundation)
  find_library(IOKIT IOKit)
  find_library(APPKIT AppKit)
  list(APPEND EXTRA_LIBRARIES ${COREFOUNDATION} ${IOKIT} ${APPKIT})
elseif(WIN32)
  list(APPEND EXTRA_LIBRARIES setupapi)
endif()

##########################
# JNI library setup
##########################
set(MONERO_JNI_SRC_FILES src/main/cpp/monero_jni_bridge.cpp)
add_library(monero-java SHARED ${MONERO_JNI_SRC_FILES})

target_include_directories(monero-java PUBLIC
  "$ENV{JAVA_HOME}"
  "$ENV{JAVA_HOME}/include"
  "${Boost_INCLUDE_DIR}"
  "${OPENSSL_INCLUDE_DIR}"
  "${MONERO_CPP_DIR}/external/libsodium/include/sodium"
  "${MONERO_CPP_DIR}/external/openssl-sdk/include"
  "${MONERO_CPP_DIR}/src"
  "${MONERO_PROJECT_DIR}/contrib/epee/include"
  "${MONERO_PROJECT_DIR}/external"
  "${MONERO_PROJECT_DIR}/external/easylogging++"
  "${MONERO_PROJECT_DIR}/external/rapidjson/include"
  "${MONERO_PROJECT_DIR}/src"
  "${MONERO_PROJECT_DIR}/src/crypto"
  "${MONERO_PROJECT_DIR}/src/crypto/crypto_ops_builder/include"
  "${MONERO_PROJECT_DIR}/src/wallet"
  "${MONERO_PROJECT_DIR}/src/wallet/api"
)

if(APPLE)
  target_include_directories(monero-java PUBLIC "$ENV{JAVA_HOME}/include/darwin")
elseif(WIN32)
  target_include_directories(monero-java PUBLIC "$ENV{JAVA_HOME}/include/win32")
else()
  target_include_directories(monero-java PUBLIC "$ENV{JAVA_HOME}/include/linux")
endif()

target_link_libraries(monero-java
  monero-cpp
  wallet_api
  ${Boost_LIBRARIES}
  ${OPENSSL_LIBRARIES}
  ${EXTRA_LIBRARIES}
)

if(WIN32)
  target_link_options(monero-java PUBLIC "-Wl,--enable-auto-import,--export-all-symbols")
endif()

if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
  target_link_options(monero-java PRIVATE "-z" "noexecstack")
endif()

install(TARGETS monero-java
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR} COMPONENT Runtime
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR} COMPONENT Runtime
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR} COMPONENT Development
)

if(APPLE)
  add_custom_command(TARGET monero-java
    POST_BUILD COMMAND
    ${CMAKE_INSTALL_NAME_TOOL} -add_rpath "@loader_path/" $<TARGET_FILE:monero-java>
  )
endif()
