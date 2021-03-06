IF(MNN_BUILD_CONVERTER)
  option(TF_CONVERT_ORIGIN "Fall Back to Origin Model Converter" OFF)
  option(TFMODEL_OPTIMIZE "Enable tensorflow model optimizer" OFF)
  IF(MNN_PORTABLE_BUILD)
    SET(Protobuf_USE_STATIC_LIBS ON)
  ENDIF()
  find_package(Protobuf REQUIRED)
  if (${CMAKE_VERSION} VERSION_LESS "3.6.0")
      set(Protobuf_LIBRARIES ${PROTOBUF_LIBRARIES})
      set(Protobuf_INCLUDE_DIRS ${PROTOBUF_INCLUDE_DIRS})
  endif()
  SET(MNN_CONVERTER_INC ${CMAKE_CURRENT_LIST_DIR}/include/ ${CMAKE_CURRENT_LIST_DIR}/source/tflite/schema/ ${Protobuf_INCLUDE_DIRS})
  SET(MNN_CONVERTER_BACKENDS_TARGETS "")
  SET(MNN_CONVERTER_BACKENDS_OBJECTS "")
  include(${CMAKE_CURRENT_LIST_DIR}/source/tensorflow/CMakeLists.txt)
  include(${CMAKE_CURRENT_LIST_DIR}/source/onnx/CMakeLists.txt)
  include(${CMAKE_CURRENT_LIST_DIR}/source/caffe/CMakeLists.txt)
  include(${CMAKE_CURRENT_LIST_DIR}/source/MNN/CMakeLists.txt)
  include(${CMAKE_CURRENT_LIST_DIR}/source/optimizer/CMakeLists.txt)
  include(${CMAKE_CURRENT_LIST_DIR}/source/tflite/CMakeLists.txt)

  file(GLOB COMMON_SRC ${CMAKE_CURRENT_LIST_DIR}/source/common/*.cpp ${CMAKE_SOURCE_DIR}/3rd_party/flatbuffers/src/util.cpp)
  add_executable(MNNDump2Json ${CMAKE_CURRENT_LIST_DIR}/source/MNNDump2Json.cpp)
  add_dependencies(MNNDump2Json MNN_SCHEMA_GEN)
  add_executable(MNNConvert
    ${CMAKE_CURRENT_LIST_DIR}/source/MNNConverter.cpp
    ${CMAKE_CURRENT_LIST_DIR}/source/cli.cpp
    ${CMAKE_CURRENT_LIST_DIR}/source/config.cpp
    ${MNN_SOURCE_DIR}/3rd_party/flatbuffers/src/util.cpp
    ${SCHEMA_TARGETS}
    )
  IF(MNN_BUILD_SHARED_LIBS)
    add_library(MNNConvertDeps SHARED ${COMMON_SRC} ${MNN_CONVERTER_BACKENDS_OBJECTS} ${PROJECT_SOURCE_DIR}/cmake/dummy.cpp)
    target_link_libraries(MNNConvertDeps PRIVATE ${MNN_DEPS} ${Protobuf_LIBRARIES})
  ELSE()
    add_library(MNNConvertDeps STATIC ${COMMON_SRC} ${MNN_CONVERTER_BACKENDS_OBJECTS} ${PROJECT_SOURCE_DIR}/cmake/dummy.cpp)
    target_link_libraries(MNNConvertDeps INTERFACE ${MNN_DEPS} ${Protobuf_LIBRARIES})
  ENDIF()
  add_dependencies(MNNConvertDeps ${MNN_DEPS})
  target_link_libraries(MNNConvert ${MNN_DEPS} ${Protobuf_LIBRARIES})

  IF(NOT MNN_BUILD_SHARED_LIBS)
    if(APPLE)
        target_link_libraries(MNNConvert -Wl,-all_load MNNConvertDeps -Wl,-noall_load)
    elseif (CMAKE_CXX_COMPILER_ID MATCHES "GNU" OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
        target_link_libraries(MNNConvert -Wl,--whole-archive MNNConvertDeps -Wl,--no-whole-archive)
    endif()
  ELSE()
    target_link_libraries(MNNConvert MNNConvertDeps)
  ENDIF()
  add_dependencies(MNNConvert MNN_SCHEMA_GEN)
  FOREACH(CONVERTER_TARGET ${MNN_CONVERTER_BACKENDS_TARGETS} MNNConvert MNNDump2Json MNNConvertDeps)
    target_include_directories(${CONVERTER_TARGET} PRIVATE ${MNN_CONVERTER_INC})
    add_dependencies(${CONVERTER_TARGET} MNN_SCHEMA_GEN)
  ENDFOREACH()
ENDIF()
