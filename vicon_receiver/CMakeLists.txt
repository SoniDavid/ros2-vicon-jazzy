# Jazzy port. The ROS code is unchanged from the Humble fork; everything here is build system.
cmake_minimum_required(VERSION 3.8)
project(vicon_receiver)

if(NOT CMAKE_C_STANDARD)
  set(CMAKE_C_STANDARD 99)
endif()

# Jazzy's rclcpp requires C++17. The Humble fork's 14 only ever worked because rclcpp's
# INTERFACE cxx_std_17 won the negotiation.
if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 17)
endif()

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic -g)
endif()

# No Boost. The SDK is built against Boost 1.58 and ships its own stubs; linking this node
# against system Boost 1.83 as well puts 1.83's header-only boost::system symbols in the
# executable, where the dynamic linker lets them preempt the SDK's and it segfaults inside
# Connect(). The node now uses std::thread/std::mutex and exports no Boost symbols at all.
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(geometry_msgs REQUIRED)

# The SDK is x86-64 only: elsewhere (the Pi) build an empty package so a full workspace build passes.
if(NOT CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|AMD64)$")
  message(WARNING "vicon_receiver: Vicon SDK is x86-64 only, skipping on ${CMAKE_SYSTEM_PROCESSOR}")
  ament_package()
  return()
endif()

include_directories(include)

# The vendored Vicon SDK, by path rather than a bare -l: the Humble fork linked the bare name and
# so silently depended on install_libs.sh having copied it into /usr/lib first.
# NOTE: this .so is x86-64 ONLY. The Raspberry Pi cannot run this node; see the README.
find_library(VICON_SDK_LIB ViconDataStreamSDK_CPP
  HINTS ${CMAKE_CURRENT_SOURCE_DIR}/../DataStreamSDK_10.1 /usr/lib
  NO_DEFAULT_PATH)
if(NOT VICON_SDK_LIB)
  find_library(VICON_SDK_LIB ViconDataStreamSDK_CPP REQUIRED)
endif()
message(STATUS "vicon_receiver: Vicon DataStream SDK at ${VICON_SDK_LIB}")

add_executable(vicon_client src/vicon_receiver/communicator.cpp src/vicon_receiver/publisher.cpp)
target_link_libraries(vicon_client ${VICON_SDK_LIB})
ament_target_dependencies(vicon_client rclcpp geometry_msgs)

# The SDK has no SONAME and needs its Boost 1.58 stubs at run time; install_libs.sh puts both in
# /usr/lib. Carry the build-tree path too so a build that skipped that script still runs.
set_target_properties(vicon_client PROPERTIES
  BUILD_RPATH "${CMAKE_CURRENT_SOURCE_DIR}/../DataStreamSDK_10.1"
  INSTALL_RPATH "${CMAKE_CURRENT_SOURCE_DIR}/../DataStreamSDK_10.1")

# get_default_rmw_implementation()/get_rmw_typesupport() removed: they came from
# rmw_implementation_cmake, which was never find_package'd here, and their results were unused.

install(TARGETS vicon_client DESTINATION lib/${PROJECT_NAME})

install(DIRECTORY
  launch
  DESTINATION share/${PROJECT_NAME}/
)

ament_package()
