# ZMQPP Cmake build description.
# ==============================
#
# CMake should find the zmq libraries / headers automatically if they are
# installed system-wide. If CMake cannot find them not, or you would like to
# use custom built ones, set these variables:
#
#   - ZEROMQ_LIB_DIR to the directory where libzmq / libzmq-shared is located
#
#   - ZEROMQ_INCLUDE_DIR to the directory where zmq.h is located
#


cmake_minimum_required(VERSION 3.0)

include("cmake/HunterGate.cmake")
HunterGate(
  URL "https://github.com/ruslo/hunter/archive/v0.12.44.tar.gz"
  SHA1 "54963db4cd6e2a8780643a3cc48bdbbefedec6cc"
)

project(ZMQPP VERSION 4.1.2)
enable_testing()

hunter_add_package(ZeroMQ)
find_package(ZeroMQ CONFIG REQUIRED)
if(NOT MSVC)
  string(REGEX MATCH "-std=c[+][+]11" CXX11_PRESENTED "${CMAKE_CXX_FLAGS}")
  if(NOT CXX11_PRESENTED)
    message(FATAL_ERROR "C++11 required for building ${PROJECT_NAME}")
  endif()
endif()

# Set a consistent MACOSX_RPATH default across all CMake versions.  When CMake
# 2.8.12 is required, change this default to 1.  When CMake 3.0.0 is required,
# remove this block (see CMP0042).
#
# TODO: verify correctness of this flag
if(NOT DEFINED CMAKE_MACOSX_RPATH)
  set(CMAKE_MACOSX_RPATH 0)
endif()

set( ZMQPP_BUILD_STATIC   true   CACHE BOOL "Build the ZMQPP static library" )
set( ZMQPP_BUILD_SHARED   true   CACHE BOOL "Build the ZMQPP dynamic library" )

set( ZMQPP_BUILD_EXAMPLES false   CACHE BOOL "Build the ZMQPP examples" )
set( ZMQPP_BUILD_CLIENT   false   CACHE BOOL "Build the ZMQPP client" )
# never switch on it won't be installed.
set( ZMQPP_BUILD_TESTS    false   CACHE BOOL "Build the ZMQPP tests" )


# Since the current CMake build of ZMQ does not work for generating a dynamic libzmq,
# give a chance for users to update which ZMQ library to link to

# zmq-static is the name of the static target in libzmq's CMakeLists.txt
set( ZMQPP_LIBZMQ_NAME_STATIC  "zmq-static" CACHE STRING "The ZMQ library to link the static ZMQPP. (if built)" )
set( ZMQPP_LIBZMQ_NAME_SHARED  "zmq"        CACHE STRING "The ZMQ library to link the dynamic ZMQPP. (if built)" )

# Paths to set to look for zmq
set( ZEROMQ_LIB_DIR       ""      CACHE PATH "The library directory for libzmq" )
set( ZEROMQ_INCLUDE_DIR   ""      CACHE PATH "The include directory for ZMQ" )

# Build flags
set( IS_TRAVIS_CI_BUILD   true    CACHE bool "Defines TRAVIS_CI_BUILD - Should the tests avoid running cases where memory is scarce." )

# Do not run some tests when building on travis-ci (this cause oom error and kill the test
# process)
# These tests seem to be:
#    - sending_large_messages_string
if (IS_TRAVIS_CI_BUILD)
  add_definitions( -DTRAVIS_CI_BUILD)
endif()

set( INSTALL_TARGET_LIST )

# The library to link with the examples and the tests.
# Because we may or may not build shared/static libs, this needs to
# be dynamic
set( LIB_TO_LINK_TO_EXAMPLES )


# libzmqpp
# --------

set( LIBZMQPP_SOURCES
  src/zmqpp/actor.cpp
  src/zmqpp/context.cpp
  src/zmqpp/curve.cpp
  src/zmqpp/frame.cpp
  src/zmqpp/message.cpp
  src/zmqpp/poller.cpp
  src/zmqpp/reactor.cpp
  src/zmqpp/signal.cpp
  src/zmqpp/socket.cpp
  src/zmqpp/z85.cpp
  src/zmqpp/zap_request.cpp
  src/zmqpp/auth.cpp
  src/zmqpp/zmqpp.cpp
  )

# Staticlib
if (ZMQPP_BUILD_STATIC)
  add_library( zmqpp-static STATIC ${LIBZMQPP_SOURCES})
  target_link_libraries(zmqpp-static ZeroMQ::libzmq-static)
  list( APPEND INSTALL_TARGET_LIST zmqpp-static)
  set( LIB_TO_LINK_TO_EXAMPLES zmqpp-static )
endif() # ZMQPP_BUILD_STATIC

# Shared lib
if (ZMQPP_BUILD_SHARED)
  add_library( zmqpp SHARED ${LIBZMQPP_SOURCES})

  target_link_libraries(zmqpp ZeroMQ::libzmq)

  list( APPEND INSTALL_TARGET_LIST zmqpp)
  set( LIB_TO_LINK_TO_EXAMPLES zmqpp )
endif() # ZMQPP_BUILD_SHARED

if ((ZMQPP_BUILD_CLIENT) OR (ZMQPP_BUILD_EXAMPLES) OR (ZMQPP_BUILD_TESTS))
  hunter_add_package(Boost COMPONENTS program_options thread system test)
  find_package(Boost CONFIG REQUIRED program_options thread system unit_test_framework)

  include_directories("${CMAKE_CURRENT_SOURCE_DIR}/src")
endif()

# Examples
# --------

if(ZMQPP_BUILD_EXAMPLES)
  add_executable( zmqpp-simple-server
    examples/simple_server
    )

  add_executable( zmqpp-simple-client
    examples/simple_client
    )

  target_link_libraries( zmqpp-simple-client ${LIB_TO_LINK_TO_EXAMPLES} )
  target_link_libraries( zmqpp-simple-server ${LIB_TO_LINK_TO_EXAMPLES} )
  list( APPEND INSTALL_TARGET_LIST zmqpp-simple-server zmqpp-simple-client )
endif()

# Client
# ------
if( ZMQPP_BUILD_CLIENT )
  include_directories("${CMAKE_CURRENT_SOURCE_DIR}/src")
  add_executable( zmqpp-client
    src/client/main.cpp
    src/client/options.cpp
    )
  target_link_libraries( zmqpp-client ${LIB_TO_LINK_TO_EXAMPLES} ${Boost_LIBRARIES} )
  list( APPEND INSTALL_TARGET_LIST zmqpp-client )
endif()

# Tests
# -----

if( ZMQPP_BUILD_TESTS )
  add_executable( zmqpp-test-runner
    src/tests/test_actor.cpp
    src/tests/test_context.cpp
    src/tests/test_inet.cpp
    src/tests/test_load.cpp
    src/tests/test_message.cpp
    src/tests/test_message_stream.cpp
    src/tests/test_poller.cpp
    src/tests/test_reactor.cpp
    src/tests/test_sanity.cpp
    src/tests/test_socket.cpp
    src/tests/test_socket_options.cpp
    src/tests/test_z85.cpp
    src/tests/test_auth.cpp
    )
  target_link_libraries( zmqpp-test-runner  ${LIB_TO_LINK_TO_EXAMPLES} Boost::thread Boost::system Boost::unit_test_framework)
  add_test( zmqpp-test zmqpp-test-runner --log-level=test-suite )
endif()


# Install
# -------
set(config_install_dir "lib/cmake/${PROJECT_NAME}")
set(config_export_name "${PROJECT_NAME}Config")
set(targets_export_name "${PROJECT_NAME}Targets")
set(project_config "${CMAKE_CURRENT_BINARY_DIR}/${config_export_name}.cmake")
set(version_config "${CMAKE_CURRENT_BINARY_DIR}/${config_export_name}Version.cmake")
set(namespace "${PROJECT_NAME}::")

install(TARGETS ${INSTALL_TARGET_LIST}
        EXPORT "${targets_export_name}"
        INCLUDES DESTINATION include
        RUNTIME DESTINATION bin
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib)

install(DIRECTORY src/zmqpp DESTINATION include/
        FILES_MATCHING PATTERN "*.hpp")


include(CMakePackageConfigHelpers)
configure_package_config_file( # Uses target_exports_name
    "cmake/Config.cmake.in"
    "${project_config}"
    INSTALL_DESTINATION "${config_install_dir}"
)
write_basic_package_version_file(
  "${version_config}" COMPATIBILITY SameMajorVersion)

install(
  EXPORT "${targets_export_name}"
  NAMESPACE "${namespace}"
  DESTINATION "${config_install_dir}")

 install(
   FILES "${project_config}" "${version_config}"
   DESTINATION "${config_install_dir}")
