cmake_minimum_required(VERSION 2.8.12 FATAL_ERROR)

# ---[ Project and semantic versioning.
project(Caffe2 CXX C)

set(CAFFE2_VERSION_MAJOR 0)
set(CAFFE2_VERSION_MINOR 8)
set(CAFFE2_VERSION_PATCH 1)
set(CAFFE2_VERSION
    ${CAFFE2_VERSION_MAJOR}.${CAFFE2_VERSION_MINOR}.${CAFFE2_VERSION_PATCH})


# ---[ Do not search cmake install prefix. See the following link for details:
# https://github.com/caffe2/caffe2/commit/251d2d0c521c781276eea3748d92fc92e5dbceab
message(STATUS "Setting CMAKE_FIND_NO_INSTALL_PREFIX")
set(CMAKE_FIND_NO_INSTALL_PREFIX TRUE)


# ---[ Options.
# Note to developers: if you add an option below, make sure you also add it to
# cmake/Summary.cmake so that the summary prints out the option values.
option(BUILD_BINARY "Build C++ binaries" ON)
option(BUILD_PYTHON "Build Python binaries" ON)
option(BUILD_SHARED_LIBS "Build libcaffe2.so" ON)
option(BUILD_SHARE_DIR "Build files in the share/ directory" OFF)
option(BUILD_TEST "Build C++ test binaries (need gtest and gbenchmark)" ON)
option(USE_ATEN "Use ATen" OFF)
option(USE_ASAN "Use Address Sanitizer" OFF)
option(USE_CUDA "Use Cuda" ON)
option(USE_FFMPEG "Use ffmpeg" OFF)
option(USE_GFLAGS "Use GFLAGS" ON)
option(USE_GLOG "Use GLOG" ON)
option(USE_GLOO "Use Gloo" ON)
option(USE_LEVELDB "Use LEVELDB" ON)
option(USE_LITE_PROTO "Use lite protobuf instead of full." OFF)
option(USE_LMDB "Use LMDB" ON)
option(USE_METAL "Use Metal for iOS build" ON)
option(USE_MOBILE_OPENGL "Use OpenGL for mobile code" ON)
option(USE_MPI "Use MPI" ON)
option(USE_NCCL "Use NCCL" ON)
option(USE_NERVANA_GPU "Use Nervana GPU backend" OFF)
option(USE_NNPACK "Use NNPACK" ON)
option(USE_OBSERVERS "Use Observer Library" OFF)
option(USE_OPENCV "Use openCV" ON)
option(USE_OPENMP "Use OpenMP for parallel code" ON)
option(USE_REDIS "Use Redis" OFF)
option(USE_ROCKSDB "Use RocksDB" ON)
option(USE_SNPE "Use Qualcomm's SNPE library" OFF)
option(USE_THREADS "Use Threads" ON)
option(USE_ZMQ "Use ZMQ" OFF)


# ---[ CMake scripts + modules
list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake/Modules)
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
set(CMAKE_MACOSX_RPATH 1)

enable_testing()

# External projects
include(ExternalProject)

include(cmake/Utils.cmake)
include(cmake/Summary.cmake)

set(CAFFE2_WHITELIST "" CACHE STRING "A whitelist file of files that one should build.")

# Set default build type
if(NOT CMAKE_BUILD_TYPE)
    message(STATUS "Build type not set - defaulting to Release")
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Choose the type of build from: Debug Release RelWithDebInfo MinSizeRel Coverage." FORCE)
endif()

# ---[ Dependencies
include(cmake/Dependencies.cmake)

# ---[ Misc checks to cope with various compiler modes
include(cmake/MiscCheck.cmake)

# ---[ Whitelist file if whitelist is specified
include(cmake/Whitelist.cmake)

# ---[ Set link flag, handle additional deps for gcc 4.8 and above
if(CMAKE_COMPILER_IS_GNUCXX AND CMAKE_CXX_COMPILER_VERSION VERSION_GREATER 4.8.0 AND NOT ANDROID)
  message(STATUS "GCC ${CMAKE_CXX_COMPILER_VERSION}: Adding gcc and gcc_s libs to link line")
  list(APPEND Caffe2_DEPENDENCY_LIBS gcc_s gcc)
endif()

# ---[ Set output directories
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "binaries")

# ---[ Build flags
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=c11")
if(NOT MSVC)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -O2 -fPIC")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-narrowing")
else()
  foreach(flag_var
      CMAKE_CXX_FLAGS CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE
      CMAKE_CXX_FLAGS_MINSIZEREL CMAKE_CXX_FLAGS_RELWITHDEBINFO)
    if (NOT ${BUILD_SHARED_LIBS})
      if(${flag_var} MATCHES "/MD")
        string(REGEX REPLACE "/MD" "/MT" ${flag_var} "${${flag_var}}")
      endif(${flag_var} MATCHES "/MD")
    endif()
    set(${flag_var} "${${flag_var}} /MP /bigobj")
  endforeach(flag_var)
endif()

if(NOT APPLE AND UNIX)
  list(APPEND Caffe2_DEPENDENCY_LIBS dl)
endif()

# Prefix path to Caffe2 headers.
# If a directory containing installed Caffe2 headers was inadvertently
# added to the list of include directories, prefixing
# PROJECT_SOURCE_DIR means this source tree always takes precedence.
include_directories(BEFORE ${PROJECT_SOURCE_DIR})

# Prefix path to generated Caffe2 headers.
# These need to take precedence over their empty counterparts located
# in PROJECT_SOURCE_DIR.
include_directories(BEFORE ${PROJECT_BINARY_DIR})

# ---[ Old caffe protobuf.
add_subdirectory(caffe/proto)

# ---[ Main build
add_subdirectory(caffe2)

caffe2_print_configuration_summary()

if(NOT TARGET uninstall)
  configure_file(
      ${CMAKE_CURRENT_SOURCE_DIR}/cmake/cmake_uninstall.cmake.in
      ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake
      IMMEDIATE @ONLY)

  add_custom_target(uninstall
      COMMAND ${CMAKE_COMMAND} -P
      ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)
endif()
