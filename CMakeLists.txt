cmake_minimum_required(VERSION 3.20.0)
# toolchain need to be set before project
set(CMAKE_TOOLCHAIN_FILE "/home/lzy/vcpkg/scripts/buildsystems/vcpkg.cmake" CACHE STRING "Vcpkg toolchain file")
set(CMAKE_CXX_COMPILER "/usr/bin/clang++-18")
set(CMAKE_C_COMPILER "/usr/bin/clang-18")
project(demo VERSION 0.0.1 LANGUAGES C CXX)
set(CMAKE_BUILD_PARALLEL_LEVEL 50)
# ------ Standard ------
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# ------ Build tool ------
set(CMAKE_VERBOSE_MAKEFILE ON)
set(CMAKE_GENERATOR "Unix Makefiles")

# ------ Build type ------
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE "RelWithDebInfo") # default build type
endif()
message(STATUS "CMAKE_BUILD_TYPE: ${CMAKE_BUILD_TYPE}")
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
  add_compile_definitions(__DEBUG__) # set __DEBUG__ for debug release
endif()
if(CMAKE_BUILD_TYPE STREQUAL "RelWithDebInfo")
  add_compile_definitions(__DEBUG__) # set __DEBUG__ for debug release
endif()
if(CMAKE_BUILD_TYPE STREQUAL "RelWithSizeOp")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Oz -pipe -fno-math-errno -flto=auto")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Oz -pipe -fno-math-errno -flto=auto")
endif()
if(CMAKE_BUILD_TYPE STREQUAL "Release")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -pipe -fno-math-errno -flto=auto")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pipe -fno-math-errno -flto=auto")
endif()

# Export compile_commands.json
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(CMAKE_CXX_CLANG_TIDY "clang-tidy")
set(BUILD_SUPPORT_DIR "${CMAKE_SOURCE_DIR}/build_support")

# clang-format
find_program(CLANG_FORMAT_BIN
        NAMES clang-format clang-format-14)
# clang-tidy        
find_program(CLANG_TIDY_BIN
        NAMES clang-tidy clang-tidy-14)
# cpplint
find_program(CPPLINT_BIN
        NAMES cpplint cpplint.py
        HINTS "${BUILD_SUPPORT_DIR}")
# #########################################
# "make format"
# "make check-format"
# #########################################
string(CONCAT FORMAT_DIRS
        "${CMAKE_CURRENT_SOURCE_DIR}/src,"
        # "${CMAKE_CURRENT_SOURCE_DIR}/test/MapReduce,"
)

# Runs clang format and updates files in place.
add_custom_target(format ${BUILD_SUPPORT_DIR}/run_clang_format.py
        ${CLANG_FORMAT_BIN}
        ${BUILD_SUPPORT_DIR}/clang_format_exclusions.txt
        --source_dirs
        ${FORMAT_DIRS}
        --fix
        --quiet
)

# Runs clang format and exits with a non-zero exit code if any files need to be reformatted
add_custom_target(check-format ${BUILD_SUPPORT_DIR}/run_clang_format.py
        ${CLANG_FORMAT_BIN}
        ${BUILD_SUPPORT_DIR}/clang_format_exclusions.txt
        --source_dirs
        ${FORMAT_DIRS}
        --quiet
)

# #########################################
# "make check-lint"
# #########################################
file(GLOB_RECURSE LINT_FILES
        "${CMAKE_CURRENT_SOURCE_DIR}/src/*.h"
        "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/test/MapReduce/*.h"
        "${CMAKE_CURRENT_SOURCE_DIR}/test/MapReduce/*.cpp"
)

# Balancing act: cpplint.py takes a non-trivial time to launch,
# so process 12 files per invocation, while still ensuring parallelism
add_custom_target(check-lint echo '${LINT_FILES}' | xargs -n12 -P8
        ${CPPLINT_BIN}
        --verbose=2 --quiet
        --linelength=120
        --filter=-legal/copyright,-build/header_guard,-runtime/references # https://github.com/cpplint/cpplint/issues/148
)

# ##########################################################
# "make check-clang-tidy" target
# ##########################################################
# runs clang-tidy and exits with a non-zero exit code if any errors are found.
# note that clang-tidy automatically looks for a .clang-tidy file in parent directories
add_custom_target(check-clang-tidy
        ${BUILD_SUPPORT_DIR}/run_clang_tidy.py # run LLVM's clang-tidy script
        -clang-tidy-binary ${CLANG_TIDY_BIN} # using our clang-tidy binary
        -p ${CMAKE_BINARY_DIR} # using cmake's generated compile commands
)
add_custom_target(fix-clang-tidy
        ${BUILD_SUPPORT_DIR}/run_clang_tidy.py # run LLVM's clang-tidy script
        -clang-tidy-binary ${CLANG_TIDY_BIN} # using our clang-tidy binary
        -p ${CMAKE_BINARY_DIR} # using cmake's generated compile commands
        -clang-apply-replacements-binary ${CLANG_APPLY_REPLACEMENTS_BIN} # using our clang-apply-replacements binary
        -fix # apply suggested changes generated by clang-tidy
)
add_custom_target(check-clang-tidy-diff
        ${BUILD_SUPPORT_DIR}/run_clang_tidy.py # run LLVM's clang-tidy script
        -clang-tidy-binary ${CLANG_TIDY_BIN} # using our clang-tidy binary
        -p ${CMAKE_BINARY_DIR} # using cmake's generated compile commands
        -only-diff # only check diff files to master
)
add_custom_target(fix-clang-tidy-diff
        ${BUILD_SUPPORT_DIR}/run_clang_tidy.py # run LLVM's clang-tidy script
        -clang-tidy-binary ${CLANG_TIDY_BIN} # using our clang-tidy binary
        -p ${CMAKE_BINARY_DIR} # using cmake's generated compile commands
        -clang-apply-replacements-binary ${CLANG_APPLY_REPLACEMENTS_BIN} # using our clang-apply-replacements binary
        -fix # apply suggested changes generated by clang-tidy
        -only-diff # only check diff files to master
)


set(CMAKE_BUILD_TYPE "Debug")
set(CMAKE_CXX_FLAGS_DEBUG "$ENV{CXXFLAGS} -O0 -Wall -g -ggdb")
set(CMAKE_CXX_FLAGS_RELEASE "$ENV{CXXFLAGS} -DNODEBUG -O3 -Wall")

############# 3rd ############
# # absl
# find_package(absl REQUIRED)
# # GRPC库
# # set(CMAKE_PREFIX_PATH "/home/lzy/grpc" ${CMAKE_PREFIX_PATH})
# find_package(Protobuf REQUIRED)
# find_package(gRPC REQUIRED)
# set(PROTOBUF3_PROTOC3_EXECUTABLE "/home/lzy/vcpkg/installed/x64-linux/tools/protobuf/protoc")
# set(PROTOC_GEN_GRPC_CPP_PLUGIN "/home/lzy/vcpkg/installed/x64-linux/tools/grpc/grpc_cpp_plugin")
# set(_PROTOBUF_LIBPROTOBUF protobuf::libprotobuf)
# set(_REFLECTION gRPC::grpc++_reflection)
# set(_GRPC_GRPCPP gRPC::grpc++)


# ### 处理proto ### 
# set(PROTOS_DIR "${CMAKE_SOURCE_DIR}/proto")
# file(GLOB PROTO_FILES "${PROTOS_DIR}/*.proto")
# set(proto_srcs "")
# set(proto_hdrs "")
# set(grpc_srcs "")
# set(grpc_hdrs "")

# foreach(proto_file ${PROTO_FILES})
#   # 生成每个.proto文件的源文件路径
#   get_filename_component(proto_name ${proto_file} NAME_WE)  # 去掉路径和扩展名

#   list(APPEND proto_srcs "${CMAKE_SOURCE_DIR}/protohelper/${proto_name}.pb.cc")
#   list(APPEND proto_hdrs "${CMAKE_SOURCE_DIR}/protohelper/${proto_name}.pb.h")
#   list(APPEND grpc_srcs "${CMAKE_SOURCE_DIR}/protohelper/${proto_name}.grpc.pb.cc")
#   list(APPEND grpc_hdrs "${CMAKE_SOURCE_DIR}/protohelper/${proto_name}.grpc.pb.h")
# endforeach()

# add_custom_command(
#   OUTPUT ${proto_srcs} ${proto_hdrs} ${grpc_srcs} ${grpc_hdrs}
#   COMMAND ${PROTOBUF3_PROTOC3_EXECUTABLE}
#   ARGS --grpc_out "${CMAKE_SOURCE_DIR}/protohelper"
#        --cpp_out "${CMAKE_SOURCE_DIR}/protohelper"
#        -I "${PROTOS_DIR}"
#        --plugin=protoc-gen-grpc="${PROTOC_GEN_GRPC_CPP_PLUGIN}"
#        ${PROTO_FILES}  # 传入多个proto文件
#   DEPENDS ${PROTO_FILES}  # 依赖多个proto文件
# )

# add_library(GRPC_PROTO
#   ${grpc_srcs}
#   ${grpc_hdrs}
#   ${proto_srcs}
#   ${proto_hdrs})
  
# target_link_libraries(GRPC_PROTO
#   ${_REFLECTION}
#   ${_GRPC_GRPCPP}
#   ${_PROTOBUF_LIBPROTOBUF})

# set(GRPC_LIB GRPC_PROTO)

############# include ##############
include_directories(${PROJECT_SOURCE_DIR}/include)

############# subdirectory ##############
add_subdirectory(./src)

add_subdirectory(./test)

add_subdirectory(./example)

