cmake_minimum_required (VERSION 3.1.0)

set(PROJECT_NAME COLLADA2GLTF)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_EXTENSIONS OFF)
include(ExternalProject)

project(${PROJECT_NAME})

# cmake -Dtest=ON to build with tests
option(test "Build all tests." OFF)

# GLTF
include_directories(GLTF/include)
add_subdirectory(GLTF)

# Disable for other modules
set(TEST_ENABLED ${test})
set(test OFF)

# RapidJSON
if(DEFINED RAPIDJSON_INCLUDE_DIR)
  include_directories(${RAPIDJSON_INCLUDE_DIR})
else()
  include_directories(GLTF/dependencies/rapidjson/include)
endif()

# Draco
find_package(draco 1.5.2 CONFIG HINTS "${DRACO_INSTALL_PREFIX}")
if(NOT TARGET draco::draco)
  if(DEFINED DRACO_INCLUDE_DIR)
    include_directories(${DRACO_INCLUDE_DIR})
  else()
    include_directories(GLTF/dependencies/draco/src)
  endif()
endif()

# Pre-compiled
if(precompiled MATCHES "X86")
  if(precompiled MATCHES "DEBUG")
    if(MSVC)
      ExternalProject_Add(
        precompiled
        URL https://github.com/KhronosGroup/COLLADA2GLTF/releases/download/dependencies/dependencies-windows-Debug-Win32.zip
        CONFIGURE_COMMAND ""
        BUILD_COMMAND ""
        INSTALL_COMMAND ""
      )
      set(PRECOMPILED_DIR ${CMAKE_CURRENT_BINARY_DIR}/precompiled-prefix/src/precompiled)
    endif()
  elseif(precompiled MATCHES "RELEASE")
    if(MSVC)
      ExternalProject_Add(
        precompiled
        URL https://github.com/KhronosGroup/COLLADA2GLTF/releases/download/dependencies/dependencies-windows-Release-Win32.zip
        CONFIGURE_COMMAND ""
        BUILD_COMMAND ""
        INSTALL_COMMAND ""
      )
      set(PRECOMPILED_DIR ${CMAKE_CURRENT_BINARY_DIR}/precompiled-prefix/src/precompiled)
    endif()
  endif()
elseif(precompiled MATCHES "X64")
  if(precompiled MATCHES "DEBUG")
    if(MSVC)
      ExternalProject_Add(
        precompiled
        URL https://github.com/KhronosGroup/COLLADA2GLTF/releases/download/dependencies/dependencies-windows-Debug-x64.zip
        CONFIGURE_COMMAND ""
        BUILD_COMMAND ""
        INSTALL_COMMAND ""
      )
      set(PRECOMPILED_DIR ${CMAKE_CURRENT_BINARY_DIR}/precompiled-prefix/src/precompiled)
    endif()
  elseif(precompiled MATCHES "RELEASE")
    if(MSVC)
      ExternalProject_Add(
        precompiled
        URL https://github.com/KhronosGroup/COLLADA2GLTF/releases/download/dependencies/dependencies-windows-Release-x64.zip
        CONFIGURE_COMMAND ""
        BUILD_COMMAND ""
        INSTALL_COMMAND ""
      )
      set(PRECOMPILED_DIR ${CMAKE_CURRENT_BINARY_DIR}/precompiled-prefix/src/precompiled)
    endif()
  endif()
endif()

if(PRECOMPILED_DIR)
  set(OpenCOLLADA
    ${PRECOMPILED_DIR}/COLLADABaseUtils.lib
    ${PRECOMPILED_DIR}/COLLADAFramework.lib
    ${PRECOMPILED_DIR}/COLLADASaxFrameworkLoader.lib
    ${PRECOMPILED_DIR}/expat.lib
    ${PRECOMPILED_DIR}/GeneratedSaxParser.lib
    ${PRECOMPILED_DIR}/LibXML.lib
    ${PRECOMPILED_DIR}/MathMLSolver.lib
    ${PRECOMPILED_DIR}/pcre.lib)
else()
  message(STATUS "No Precompiled OpenCOLLADA")
  if(DEFINED OpenCOLLADA_PREFIX AND NOT DEFINED OPENCOLLADA_VERSION_STRING)
    message(STATUS "Searching in ${OpenCOLLADA_PREFIX} because no OPENCOLLADA_VERSION_STRING")
    list(APPEND CMAKE_MODULE_PATH "${OpenCOLLADA_PREFIX}/share/opencollada/cmake/Modules")
    find_package(LibXml2 REQUIRED)
    find_package(PCRE REQUIRED) # transitive depdendency for OpenCOLLADA
    find_package(OpenCOLLADA REQUIRED CONFIG HINTS ${OpenCOLLADA_PREFIX}/lib/opencollada/cmake NO_DEFAULT_PATH)
  endif()
  if(DEFINED OPENCOLLADA_VERSION_STRING)
    message(STATUS "Have OpenCOLLADA ${OPENCOLLADA_VERSION_STRING}")
	  set(OpenCOLLADA OpenCOLLADASaxFrameworkLoader_shared)
  endif()
endif()


if(NOT DEFINED OPENCOLLADA_INCLUDE_DIRS)
# COLLADASaxFrameworkLoader/BaseUtils/Framework, GeneratedSaxParser
include_directories(dependencies/OpenCOLLADA/OpenCOLLADA/COLLADASaxFrameworkLoader/include)
include_directories(dependencies/OpenCOLLADA/OpenCOLLADA/COLLADABaseUtils/include)
include_directories(dependencies/OpenCOLLADA/OpenCOLLADA/COLLADAFramework/include)
include_directories(dependencies/OpenCOLLADA/OpenCOLLADA/GeneratedSaxParser/include)
include_directories(dependencies/OpenCOLLADA/OpenCOLLADA/Externals/pcre/include)
endif()

if(NOT OpenCOLLADA)
  add_subdirectory(dependencies/OpenCOLLADA/modules/COLLADASaxFrameworkLoader)
  set(OpenCOLLADA COLLADASaxFrameworkLoader)
endif()


# COLLADA2GLTF
include_directories(include)
file(GLOB LIB_HEADERS "include/*.h")
set(LIB_SOURCES src/COLLADA2GLTFWriter.cpp src/COLLADA2GLTFExtrasHandler.cpp)
add_library(${PROJECT_NAME} ${LIB_HEADERS} ${LIB_SOURCES})
target_link_libraries(${PROJECT_NAME} GLTF ${OpenCOLLADA})

# ahoy
include_directories(dependencies/ahoy/include)
add_subdirectory(dependencies/ahoy)

if(NOT NO_COLLADA2GLTF_BIN)
  add_executable(${PROJECT_NAME}-bin src/main.cpp)
  target_link_libraries(${PROJECT_NAME}-bin ${PROJECT_NAME} ahoy)
  if(TARGET draco::draco)
    target_link_libraries(${PROJECT_NAME}-bin draco::draco)
  else()
    target_link_libraries(${PROJECT_NAME}-bin draco)
  endif()
  install(TARGETS ${PROJECT_NAME}-bin)
endif()

if(TEST_ENABLED)
  set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
  enable_testing()

  # gtest
  set(GTEST_LINKED 1)
  include_directories(GLTF/dependencies/googletest/googletest/include)
  add_subdirectory(GLTF/dependencies/googletest/googletest)

  # Unit Tests
  include_directories(test/include)
  file(GLOB TEST_HEADERS "test/include/*.h")
  file(GLOB TEST_SOURCES "test/src/*.cpp")

  add_executable(${PROJECT_NAME}-test ${TEST_HEADERS} ${TEST_SOURCES})
  target_link_libraries(${PROJECT_NAME}-test ${PROJECT_NAME} GLTF gtest)

  add_test(COLLADA2GLTFWriterTest ${PROJECT_NAME}-test)
endif()
