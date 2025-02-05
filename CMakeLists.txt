cmake_minimum_required(VERSION 2.8)
project(HFNet_SLAM)

set(DEFAULT_BUILD_TYPE "Release")
if (NOT CMAKE_BUILD_TYPE)
    message(STATUS "Setting build type to '${DEFAULT_BUILD_TYPE}' as none was specified.")
    set(CMAKE_BUILD_TYPE "${DEFAULT_BUILD_TYPE}" CACHE STRING "Choose the type of build." FORCE)
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif ()
MESSAGE("build type: " ${CMAKE_BUILD_TYPE})
set(CMAKE_CXX_FLAGS "-std=c++14 -Wall -fopenmp -march=native -pthread")
set(CMAKE_CXX_FLAGS_DEBUG  "${CMAKE_CXX_FLAGS} -g")
set(CMAKE_CXX_FLAGS_RELEASE  "${CMAKE_CXX_FLAGS} -O3 -DNDEBUG")
# ADD_DEFINITIONS(-DREGISTER_TIMES)

LIST(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake_modules)

set(USE_TENSORRT ON)
set(USE_TENSORFLOW OFF)
set(USE_OPENVINO OFF) # HFNet-SLAM with openVINO has bad performance. So, openVINO is disabled while the code is still keep.

# use the default opencv in ubuntu 20.04
set(OpenCV_DIR /usr/lib/x86_64-linux-gnu/cmake/opencv4)
find_package(OpenCV 4.2.0 REQUIRED)
find_package(Eigen3 3.1.0 REQUIRED)
find_package(Pangolin 0.6 REQUIRED)

include_directories(
${PROJECT_SOURCE_DIR}
${PROJECT_SOURCE_DIR}/include
${PROJECT_SOURCE_DIR}/include/CameraModels
${PROJECT_SOURCE_DIR}/Thirdparty/Sophus
${EIGEN3_INCLUDE_DIR}
${OpenCV_INCLUDE_DIRS}
${Pangolin_INCLUDE_DIRS}
)

set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/lib)

add_library(${PROJECT_NAME} SHARED
src/System.cc
src/Tracking.cc
src/LocalMapping.cc
src/LoopClosing.cc
src/Extractors/BaseModel.cc
src/Extractors/HFextractor.cc
src/Extractors/HFNetTFModel.cc
src/Extractors/HFNetTFModelV2.cc
src/Extractors/HFNetRTModel.cc
src/Extractors/HFNetVINOModel.cc
src/Matcher.cc
src/FrameDrawer.cc
src/Converter.cc
src/MapPoint.cc
src/KeyFrame.cc
src/Atlas.cc
src/Map.cc
src/MapDrawer.cc
src/Optimizer.cc
src/Frame.cc
src/KeyFrameDatabase.cc
src/Sim3Solver.cc
src/Viewer.cc
src/ImuTypes.cc
src/G2oTypes.cc
src/CameraModels/Pinhole.cpp
src/CameraModels/KannalaBrandt8.cpp
src/OptimizableTypes.cpp
src/MLPnPsolver.cpp
src/GeometricTools.cc
src/TwoViewReconstruction.cc
src/Config.cc
src/Settings.cc
include/System.h
include/Tracking.h
include/LocalMapping.h
include/LoopClosing.h
include/Extractors/HFextractor.h
include/Extractors/BaseModel.h
include/Extractors/HFNetTFModel.h
include/Extractors/HFNetTFModelV2.h
include/Extractors/HFNetRTModel.h
include/Extractors/HFNetVINOModel.h
include/Matcher.h
include/FrameDrawer.h
include/Converter.h
include/MapPoint.h
include/KeyFrame.h
include/Atlas.h
include/Map.h
include/MapDrawer.h
include/Optimizer.h
include/Frame.h
include/KeyFrameDatabase.h
include/Sim3Solver.h
include/Viewer.h
include/ImuTypes.h
include/G2oTypes.h
include/CameraModels/GeometricCamera.h
include/CameraModels/Pinhole.h
include/CameraModels/KannalaBrandt8.h
include/OptimizableTypes.h
include/MLPnPsolver.h
include/GeometricTools.h
include/TwoViewReconstruction.h
include/SerializationUtils.h
include/Config.h
include/Settings.h
include/utility_common.h)

# add_subdirectory(Thirdparty/g2o)

target_link_libraries(${PROJECT_NAME}
${OpenCV_LIBS}
${EIGEN3_LIBS}
${Pangolin_LIBRARIES}
${PROJECT_SOURCE_DIR}/Thirdparty/g2o/lib/libg2o.so
-lboost_serialization
-lcrypto
)

if(${USE_TENSORRT})
    ADD_DEFINITIONS(-DUSE_TENSORRT)

    find_package(CUDA REQUIRED)

    include_directories(${TENSORRT_INCLUDE_DIR} ${CUDA_INCLUDE_DIRS})
    target_link_libraries(${PROJECT_NAME} nvinfer nvonnxparser ${CUDA_LIBRARIES})
endif()

if(${USE_TENSORFLOW})
    ADD_DEFINITIONS(-DUSE_TENSORFLOW)

    set(Tensorflow_Root "~/src/tensorflow_cc-2.9.0/tensorflow_cc/install")
    set(Tensorflow_INCLUDE_DIRS
        ${Tensorflow_Root}/include/tensorflow/bazel-bin/tensorflow/include
        ${Tensorflow_Root}/include/tensorflow/bazel-bin/tensorflow/include/src)
    set(Tensorflow_LIBS
        ${Tensorflow_Root}/lib/libtensorflow_cc.so
        ${Tensorflow_Root}/lib/libtensorflow_framework.so)

    include_directories(${Tensorflow_INCLUDE_DIRS})
    target_link_libraries(${PROJECT_NAME} ${Tensorflow_LIBS})
endif()

if(${USE_OPENVINO})
    ADD_DEFINITIONS(-DUSE_OPENVINO)

    set(OpenVINO_DIR "~/src/openvino-2022.1/install/runtime/cmake")
    find_package(OpenVINO REQUIRED)

    target_link_libraries(openvino::runtime)
endif()

# Build examples

# Utility examples
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/Examples/Utility)

if(${USE_TENSORFLOW})
add_executable(test_hfnet_tf_v1
        Examples/Utility/test_hfnet_tf_v1.cc)
target_link_libraries(test_hfnet_tf_v1 ${PROJECT_NAME})

add_executable(test_hfnet_tf_v2
        Examples/Utility/test_hfnet_tf_v2.cc)
target_link_libraries(test_hfnet_tf_v2 ${PROJECT_NAME})
endif()

if(${USE_OPENVINO})
add_executable(test_hfnet_vino
        Examples/Utility/test_hfnet_vino.cc)
target_link_libraries(test_hfnet_vino ${PROJECT_NAME})
endif()

add_executable(test_extractors
        Examples/Utility/test_extractors.cc)
target_link_libraries(test_extractors ${PROJECT_NAME})

add_executable(test_match_global_feats
        Examples/Utility/test_match_global_feats.cc)
target_link_libraries(test_match_global_feats ${PROJECT_NAME})

add_executable(test_match_local_feats
        Examples/Utility/test_match_local_feats.cc)
target_link_libraries(test_match_local_feats ${PROJECT_NAME})

# Monocular examples
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/Examples/Monocular)

add_executable(mono_euroc
        Examples/Monocular/mono_euroc.cc)
target_link_libraries(mono_euroc ${PROJECT_NAME})

add_executable(mono_tum_vi
        Examples/Monocular/mono_tum_vi.cc)
target_link_libraries(mono_tum_vi ${PROJECT_NAME})

# Monocular inertial examples
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/Examples/Monocular-Inertial)

add_executable(mono_inertial_euroc
        Examples/Monocular-Inertial/mono_inertial_euroc.cc)
target_link_libraries(mono_inertial_euroc ${PROJECT_NAME})

add_executable(mono_inertial_tum_vi
        Examples/Monocular-Inertial/mono_inertial_tum_vi.cc)
target_link_libraries(mono_inertial_tum_vi ${PROJECT_NAME})

# RGB-D examples
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/Examples/RGB-D)

add_executable(rgbd_tum
        Examples/RGB-D/rgbd_tum.cc)
target_link_libraries(rgbd_tum ${PROJECT_NAME})