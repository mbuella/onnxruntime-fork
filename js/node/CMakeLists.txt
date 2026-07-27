cmake_minimum_required(VERSION 3.11)

project(onnxruntime-node)

set(CMAKE_CXX_STANDARD 20)

# ORT_API_MANUAL_INIT is conditionally enabled based on how the core ORT libs were built
# (the --cmake_extra_defines onnxruntime_ORT_API_MANUAL_INIT=ON flag controls this)
if(onnxruntime_ORT_API_MANUAL_INIT)
  add_compile_definitions(ORT_API_MANUAL_INIT)
endif()

add_compile_definitions(NAPI_VERSION=${napi_build_version})

# dist variables
execute_process(COMMAND node -e "console.log(process.platform)"
                OUTPUT_VARIABLE node_platform OUTPUT_STRIP_TRAILING_WHITESPACE)
file(READ ${CMAKE_SOURCE_DIR}/../../VERSION_NUMBER ort_version)
string(STRIP "${ort_version}" ort_version)
set(dist_folder "${CMAKE_SOURCE_DIR}/bin/napi-v6/${node_platform}/${NODE_ARCH}/")

# onnxruntime.dll dir
if(NOT ONNXRUNTIME_BUILD_DIR)
  if(WIN32)
    set(ONNXRUNTIME_BUILD_DIR ${CMAKE_SOURCE_DIR}/../../build/Windows/${CMAKE_BUILD_TYPE})
    # ============================================================
    # Normalize to absolute path — critical so CMake computes
    # correct relative paths from .vcxproj to build output
    # ============================================================
    get_filename_component(ONNXRUNTIME_BUILD_DIR ${ONNXRUNTIME_BUILD_DIR} ABSOLUTE)
  elseif(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set(ONNXRUNTIME_BUILD_DIR ${CMAKE_SOURCE_DIR}/../../build/MacOS/${CMAKE_BUILD_TYPE})
  else()
    set(ONNXRUNTIME_BUILD_DIR ${CMAKE_SOURCE_DIR}/../../build/Linux/${CMAKE_BUILD_TYPE})
  endif()
endif()

# ============================================================
# Normalize to absolute path (outside guard — always runs)
# ============================================================
get_filename_component(ONNXRUNTIME_BUILD_DIR ${ONNXRUNTIME_BUILD_DIR} ABSOLUTE)
file(TO_CMAKE_PATH "${ONNXRUNTIME_BUILD_DIR}" ONNXRUNTIME_BUILD_DIR)

# Validate the path exists
if(NOT EXISTS "${ONNXRUNTIME_BUILD_DIR}")
  message(FATAL_ERROR "ONNXRUNTIME_BUILD_DIR does not exist: ${ONNXRUNTIME_BUILD_DIR}")
endif()

message(STATUS "onnxruntime build dir: ${ONNXRUNTIME_BUILD_DIR}")

# include dirs
include_directories(${CMAKE_JS_INC})
include_directories(${CMAKE_SOURCE_DIR}/../../include/onnxruntime/core/session)
include_directories(${CMAKE_SOURCE_DIR}/../../include/onnxruntime)
include_directories(${CMAKE_SOURCE_DIR}/../../onnxruntime)
include_directories(${CMAKE_SOURCE_DIR}/node_modules/node-addon-api)

# optional providers
option(USE_DML "Build with DirectML support" OFF)
option(USE_WEBGPU "Build with WebGPU support" OFF)
option(USE_CUDA "Build with CUDA support" OFF)
option(USE_TENSORRT "Build with TensorRT support" OFF)
option(USE_COREML "Build with CoreML support" OFF)
option(USE_QNN "Build with QNN support" OFF)

if(USE_DML)
  add_compile_definitions(USE_DML=1)
endif()
if(USE_WEBGPU)
  add_compile_definitions(USE_WEBGPU=1)
endif()
if(USE_CUDA)
  add_compile_definitions(USE_CUDA=1)
endif()
if(USE_TENSORRT)
  add_compile_definitions(USE_TENSORRT=1)
endif()
if(USE_COREML)
  add_compile_definitions(USE_COREML=1)
endif()
if(USE_QNN)
  add_compile_definitions(USE_QNN=1)
endif()

# source files
file(GLOB ORT_NODEJS_BINDING_SOURCE_FILES ${CMAKE_SOURCE_DIR}/src/*.cc)

add_library(onnxruntime_binding SHARED ${ORT_NODEJS_BINDING_SOURCE_FILES} ${CMAKE_JS_SRC})

# ============================================================
# CRITICAL: Force /MT (static runtime) at COMPILE OPTIONS level
# This has higher priority than cmake-js's CMAKE_MSVC_RUNTIME_LIBRARY
# ============================================================
if(WIN32)
  target_compile_options(onnxruntime_binding PRIVATE
    $<$<CONFIG:Debug>:/MTd>
    $<$<NOT:$<CONFIG:Debug>>:/MT>
  )
  # Also set the property as backup
  set_target_properties(onnxruntime_binding PROPERTIES
    MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>"
  )
endif()

file(MAKE_DIRECTORY ${dist_folder})

set_target_properties(onnxruntime_binding PROPERTIES
  PREFIX "" SUFFIX ".node"
  RUNTIME_OUTPUT_DIRECTORY ${dist_folder}
  RUNTIME_OUTPUT_DIRECTORY_DEBUG ${dist_folder}
  RUNTIME_OUTPUT_DIRECTORY_RELEASE ${dist_folder}
  RUNTIME_OUTPUT_DIRECTORY_Release ${dist_folder}
  LIBRARY_OUTPUT_DIRECTORY ${dist_folder}
  LIBRARY_OUTPUT_DIRECTORY_DEBUG ${dist_folder}
  LIBRARY_OUTPUT_DIRECTORY_RELEASE ${dist_folder}
  LIBRARY_OUTPUT_DIRECTORY_Release ${dist_folder}
  BUILD_WITH_INSTALL_RPATH TRUE
  INSTALL_RPATH_USE_LINK_PATH FALSE)
target_link_libraries(onnxruntime_binding PRIVATE ${CMAKE_JS_LIB})

if(MSVC AND CMAKE_JS_NODELIB_DEF AND CMAKE_JS_NODELIB_TARGET)
  execute_process(COMMAND ${CMAKE_AR} /def:${CMAKE_JS_NODELIB_DEF} /out:${CMAKE_JS_NODELIB_TARGET} ${CMAKE_STATIC_LINKER_FLAGS})
endif()

if(WIN32)
  if(${ONNXRUNTIME_GENERATOR} MATCHES "Ninja")
    set(ONNXRUNTIME_WIN_BIN_DIR ${ONNXRUNTIME_BUILD_DIR})
  else()
    set(ONNXRUNTIME_WIN_BIN_DIR ${ONNXRUNTIME_BUILD_DIR}/${CMAKE_BUILD_TYPE})
  endif()
  message(STATUS "onnxruntime dist dir: ${ONNXRUNTIME_WIN_BIN_DIR}")
endif()

if(APPLE)
  if(${ONNXRUNTIME_GENERATOR} MATCHES "Xcode")
    set(ONNXRUNTIME_MAC_BIN_DIR ${ONNXRUNTIME_BUILD_DIR}/${CMAKE_BUILD_TYPE})
  else()
    set(ONNXRUNTIME_MAC_BIN_DIR ${ONNXRUNTIME_BUILD_DIR})
  endif()
  message(STATUS "onnxruntime dist dir: ${ONNXRUNTIME_MAC_BIN_DIR}")
endif()

# ============================================================
# Link directories
# ============================================================
if(WIN32)
  target_link_directories(onnxruntime_binding PRIVATE ${ONNXRUNTIME_WIN_BIN_DIR})
elseif(APPLE)
  target_link_directories(onnxruntime_binding PRIVATE ${ONNXRUNTIME_MAC_BIN_DIR})
else()
  target_link_directories(onnxruntime_binding PRIVATE ${ONNXRUNTIME_BUILD_DIR})
endif()

# ============================================================
# Link libraries - WIN32 static build with ALL dependencies
# ============================================================
# Use GLOB_RECURSE to collect all absl .lib files automatically
# This is robust against abseil library reorganization across versions
file(GLOB_RECURSE ABSL_LIBS
  "${ONNXRUNTIME_BUILD_DIR}/_deps/abseil_cpp-build/absl/*/${CMAKE_BUILD_TYPE}/absl_*.lib")
list(SORT ABSL_LIBS)
if(WIN32)
  target_link_libraries(onnxruntime_binding PRIVATE
    # --- Protobuf (MUST be first - resolves onnx::* symbols) ---
    ${ONNXRUNTIME_BUILD_DIR}/_deps/protobuf-build/${CMAKE_BUILD_TYPE}/libprotobuf.lib
    ${ONNXRUNTIME_BUILD_DIR}/_deps/protobuf-build/${CMAKE_BUILD_TYPE}/libprotoc.lib

    ${ONNXRUNTIME_BUILD_DIR}/_deps/onnx-build/${CMAKE_BUILD_TYPE}/onnx.lib

    # --- Abseil (glob all available libs to avoid missing component errors) ---
    ${ABSL_LIBS}

    # --- re2 ---
    ${ONNXRUNTIME_BUILD_DIR}/_deps/re2-build/${CMAKE_BUILD_TYPE}/re2.lib

    # --- cpuinfo ---
    ${ONNXRUNTIME_BUILD_DIR}/_deps/pytorch_cpuinfo-build/${CMAKE_BUILD_TYPE}/cpuinfo.lib

    # --- flatbuffers ---
    ${ONNXRUNTIME_BUILD_DIR}/_deps/flatbuffers-build/${CMAKE_BUILD_TYPE}/flatbuffers.lib

    # --- MLAS dependencies (Kai kernels) ---
    ${ONNXRUNTIME_BUILD_DIR}/_deps/kleidiai-build/${CMAKE_BUILD_TYPE}/kleidiai.lib

    # --- ONNX Runtime core libraries ---
    onnxruntime_session
    onnxruntime_optimizer
    onnxruntime_providers
    onnxruntime_framework
    onnxruntime_graph
    onnxruntime_util
    onnxruntime_mlas
    onnxruntime_common
    onnxruntime_flatbuffers
    onnxruntime_providers_qnn
    onnxruntime_lora
  )
elseif(APPLE)
  target_link_libraries(onnxruntime_binding PRIVATE libonnxruntime.${ort_version}.dylib)
  set_target_properties(onnxruntime_binding PROPERTIES INSTALL_RPATH "@loader_path")
else()
  target_link_libraries(onnxruntime_binding PRIVATE onnxruntime)
  set_target_properties(onnxruntime_binding PROPERTIES INSTALL_RPATH "$ORIGIN/")
endif()

# ============================================================
# Copy QNN Backend DLLs (if static lib build)
# ============================================================
if(WIN32 AND USE_QNN)
  # Copy QnnHtp.dll (the main NPU backend)
  if(EXISTS "${ONNXRUNTIME_WIN_BIN_DIR}/QnnHtp.dll")
    file(COPY "${ONNXRUNTIME_WIN_BIN_DIR}/QnnHtp.dll" DESTINATION ${dist_folder})
    message(STATUS "Copied QnnHtp.dll to ${dist_folder}")
  endif()
endif()

# ============================================================
# Copy DLLs
# ============================================================
if(WIN32)
  if(NOT onnxruntime_BUILD_QNN_EP_STATIC_LIB)
    if(EXISTS ${ONNXRUNTIME_WIN_BIN_DIR}/onnxruntime.dll)
      file(COPY ${ONNXRUNTIME_WIN_BIN_DIR}/onnxruntime.dll DESTINATION ${dist_folder})
    endif()
  endif()
elseif(APPLE)
  file(COPY ${ONNXRUNTIME_MAC_BIN_DIR}/libonnxruntime.dylib DESTINATION ${dist_folder} FOLLOW_SYMLINK_CHAIN)
elseif(UNIX)
  file(COPY ${ONNXRUNTIME_BUILD_DIR}/libonnxruntime.so DESTINATION ${dist_folder} FOLLOW_SYMLINK_CHAIN)
else()
  message(FATAL_ERROR "Platform not supported.")
endif()

if(ORT_NODEJS_DLL_DEPS)
  foreach(dll ${ORT_NODEJS_DLL_DEPS})
    file(COPY ${dll} DESTINATION ${dist_folder})
  endforeach()
endif()
