cmake_minimum_required(VERSION 3.5)

project(euclidesdb CXX)

# ----[ Project versioning
set(EUCLIDESDB_VERSION_MAJOR 0)
set(EUCLIDESDB_VERSION_MINOR 1)
set(EUCLIDESDB_VERSION_PATCH 1)
set(EUCLIDESDB_VERSION_STRING ${EUCLIDESDB_VERSION_MAJOR}.${EUCLIDESDB_VERSION_MINOR}.${EUCLIDESDB_VERSION_PATCH})

# ----[ C++11 requirements, no gnu++11
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_COLOR_MAKEFILE ON)

Include(FindProtobuf)

# ----[ libtorch should be a symbolic link inside the building
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake")
list(APPEND CMAKE_PREFIX_PATH ./libtorch)

include(TargetCopyFiles)

# ----[ Requirements
find_package(Protobuf REQUIRED)
find_package(GRPC REQUIRED)
find_package(LevelDB REQUIRED)
find_package(Torch REQUIRED)
find_package(PythonInterp 3.6 REQUIRED)

# ---[ gRPC Protocols
set(PROTOS
    ${CMAKE_CURRENT_SOURCE_DIR}/source/protos/euclidesproto.proto
)

# ---[ This is needed due to protobuf compilation
include_directories(${CMAKE_CURRENT_BINARY_DIR})

# ---[ Protobuf and gRPC generation
protobuf_generate_cpp(PROTO_SRCS PROTO_HDRS ${PROTOS})
grpc_generate_cpp(GRPC_SRCS GRPC_HDRS ${CMAKE_CURRENT_BINARY_DIR} ${PROTOS})
protobuf_generate_python(PROTO_PY ${PROTOS})

add_custom_target(generate_proto ALL DEPENDS ${PROTO_PY})

# ----[ Set properties for generated files
set_source_files_properties(
        ${PROTO_SRCS}
        ${PROTO_HDRS}
        PROPERTIES GENERATED TRUE)

set_source_files_properties(
        ${PROTO_PY}
        PROPERTIES GENERATED TRUE)

set_source_files_properties(
        ${GRPC_SRCS}
        ${GRPC_HDRS}
        PROPERTIES GENERATED TRUE)

# ----[ Generate a header with versioning based on the template
configure_file (
        "${CMAKE_CURRENT_SOURCE_DIR}/source/EuclidesConfig.h.in"
        "${PROJECT_BINARY_DIR}/EuclidesConfig.h"
)

# ----[ Add all sources, both local and external
file(GLOB CPP_FILES source/*.cpp source/external/*.cpp)

# ----[ Copy example configuration file
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/source/external)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/source/euclidesdb.conf
               ${PROJECT_BINARY_DIR}/euclidesdb.conf COPYONLY)

# ----[ Copy all python files to the build
add_custom_target(pythongenproto ALL)
set(PYGRPC_DEST "${PROJECT_BINARY_DIR}/python")
add_custom_command(TARGET pythongenproto PRE_BUILD
        COMMAND "./gen_protobuf.sh"
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/source/python
        COMMENT "Running Python Proto generation...")

add_custom_target(pythonfiles ALL
                  DEPENDS pythongenproto)
add_copy_directory(pythonfiles source/python
        DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/python
        GLOB *.py *.jpg
        EXCLUDE *.pth *.pyc __pycache__)

# ----[ Add the main server executable
add_executable(${PROJECT_NAME}
               ${CPP_FILES}
               ${PROTO_SRCS}
               ${GRPC_SRCS})

# ----[ Add the ./lib to the RPATH for Apple and Unix
if(APPLE)
    set_target_properties(${PROJECT_NAME} PROPERTIES
        MACOSX_RPATH 1
        BUILD_WITH_INSTALL_RPATH 1
        INSTALL_RPATH "@loader_path/lib")
elseif(UNIX)
    set_target_properties(${PROJECT_NAME} PROPERTIES
            BUILD_WITH_INSTALL_RPATH 1
            INSTALL_RPATH "$ORIGIN/lib")
endif()

# ----[ Directives fro easyloggingcpp
target_compile_options(${PROJECT_NAME} PRIVATE -DELPP_FEATURE_PERFORMANCE_TRACKING -DELPP_THREAD_SAFE)

# ----[ Link it to the required dependencies
target_link_libraries(${PROJECT_NAME}
                      ${LevelDB_LIBRARIES}
                      ${TORCH_LIBRARIES}
                      gRPC::grpc++_reflection
                      protobuf::libprotobuf)

add_dependencies(${PROJECT_NAME} generate_proto)

# ----[ Copy libraries to the build directory
add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_directory
        ${CMAKE_SOURCE_DIR}/libtorch/lib $<TARGET_FILE_DIR:${PROJECT_NAME}>/lib)

install(TARGETS ${PROJECT_NAME} RUNTIME DESTINATION euclidesdb)

# ----[ Copy libtorch libraries
if(APPLE)
    install(DIRECTORY ${CMAKE_SOURCE_DIR}/libtorch/lib DESTINATION euclidesdb
            FILES_MATCHING PATTERN "*.dylib")
elseif(UNIX)
    install(DIRECTORY ${CMAKE_SOURCE_DIR}/libtorch/lib DESTINATION euclidesdb
            FILES_MATCHING PATTERN "*.so*")
endif()

# ----[ Copy the models and sample config
install(DIRECTORY ${CMAKE_SOURCE_DIR}/models DESTINATION euclidesdb
        FILES_MATCHING PATTERN "*.*" PATTERN "*.pth" EXCLUDE)

install(DIRECTORY ${CMAKE_SOURCE_DIR}/models DESTINATION euclidesdb
        FILES_MATCHING PATTERN "*.*" PATTERN "*.pth" EXCLUDE)

install(FILES ${CMAKE_SOURCE_DIR}/source/sample_config.conf DESTINATION euclidesdb)

# ----[ Packaging with CPack
execute_process(COMMAND uname -m COMMAND tr -d '\n' OUTPUT_VARIABLE ARCHITECTURE)
message("Building for ${ARCHITECTURE}")

include(InstallRequiredSystemLibraries)
set(CPACK_PACKAGE_NAME "euclidesdb")
set(CPACK_PACKAGE_VENDOR "EuclidesDB")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "EuclidesDB")
set(CPACK_PACKAGE_VERSION ${EUCLIDESDB_VERSION_STRING})
set(CPACK_PACKAGE_VERSION_MAJOR "${EUCLIDESDB_VERSION_MAJOR}")
set(CPACK_PACKAGE_VERSION_MINOR "${EUCLIDESDB_VERSION_MINOR}")
set(CPACK_PACKAGE_VERSION_PATCH "${EUCLIDESDB_VERSION_PATCH}")
set(CPACK_PACKAGE_INSTALL_DIRECTORY "euclidesdb")
set(CPACK_PACKAGE_CONTACT "christian.perone@gmail.com")

set(CPACK_GENERATOR "TGZ;ZIP;STGZ")

if(APPLE)
    set(CPACK_GENERATOR "${CPACK_GENERATOR};DragNDrop")
endif()

include(CPack)
