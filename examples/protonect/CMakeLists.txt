CMAKE_MINIMUM_REQUIRED(VERSION 2.6)

macro(auto_detect_lib LIBNAME)
  if ( PKG_CONFIG_FOUND )
    string(TOLOWER ${LIBNAME} PKGCONFIGNAME)
    pkg_check_modules(${LIBNAME} ${PKGCONFIGNAME} ${ARGN})
  else(PKG_CONFIG_FOUND)
    #MESSAGE("Fallback to non-pkg-config")
    FIND_PACKAGE(${LIBNAME} ${ARGN})
  endif(PKG_CONFIG_FOUND)
endmacro()

PROJECT(libfreenect2)
SET(CMAKE_BUILD_TYPE RelWithDebInfo)

SET(MY_DIR ${libfreenect2_SOURCE_DIR})

OPTION(ENABLE_CXX11 "Enable C++11 support" OFF)
OPTION(ENABLE_OPENCL "Enable OpenCL support" ON)
OPTION(ENABLE_OPENGL "Enable OpenGL support" ON)

IF(ENABLE_CXX11)
  INCLUDE(CheckCXXCompilerFlag)
  CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
  CHECK_CXX_COMPILER_FLAG("-std=c++0x" COMPILER_SUPPORTS_CXX0X)
  IF(COMPILER_SUPPORTS_CXX11)
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
  ELSEIF(COMPILER_SUPPORTS_CXX0X)
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
  ELSE()
    MESSAGE(STATUS "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support. Please use a different C++ compiler.")
  ENDIF()
ENDIF(ENABLE_CXX11)

# additional cmake modules
LIST(APPEND CMAKE_MODULE_PATH ${MY_DIR}/cmake_modules)

# setup threading
INCLUDE(SetupLibfreenect2Threading)

INCLUDE(GenerateResources)

#set the default path for built executables to the "bin" directory
SET(EXECUTABLE_OUTPUT_PATH ${MY_DIR}/bin)

#set the default path for built libraries to the "lib" directory
SET(LIBRARY_OUTPUT_PATH ${MY_DIR}/lib)

# dependencies
find_package(PkgConfig)    # try find PKGConfig as it will be used if found
auto_detect_lib(LibUSB-1.0 REQUIRED)
auto_detect_lib(OpenCV REQUIRED)
FIND_PACKAGE(TurboJPEG REQUIRED) #does not provide a package-config file

#hack for buggy libusb pkgconfig file
if(NOT LibUSB_INCLUDE_DIR)
  find_path(LibUSB_INCLUDE_DIR NAMES libusb.h PATHS /usr/include/libusb-1.0 )
endif()

# Add includes
INCLUDE_DIRECTORIES("${MY_DIR}/include" ${LIBFREENECT2_THREADING_INCLUDE_DIR} ${OpenCV_INCLUDE_DIRS} ${LibUSB_INCLUDE_DIR} ${TurboJPEG_INCLUDE_DIR})

if(APPLE AND NOT ${TurboJPEG_FOUND})
  # libjpeg-turbo
  INCLUDE_DIRECTORIES("${MY_DIR}/../../depends/libjpeg_turbo/include/")
  LINK_DIRECTORIES("${MY_DIR}/../../depends/libjpeg_turbo/lib/")
endif()

SET(RESOURCES_INC_FILE "${MY_DIR}/src/resources.inc")

SET(SOURCES
  include/libfreenect2/protocol/command.h
  include/libfreenect2/protocol/command_transaction.h
  include/libfreenect2/protocol/response.h
  include/libfreenect2/protocol/usb_control.h
  
  include/libfreenect2/usb/event_loop.h
  include/libfreenect2/usb/transfer_pool.h

  include/libfreenect2/async_packet_processor.h  
  include/libfreenect2/depth_packet_processor.h
  include/libfreenect2/depth_packet_stream_parser.h
  include/libfreenect2/double_buffer.h
  include/libfreenect2/frame_listener.hpp
  include/libfreenect2/frame_listener_impl.h
  include/libfreenect2/config.h
  include/libfreenect2/libfreenect2.hpp
  include/libfreenect2/packet_pipeline.h
  include/libfreenect2/packet_processor.h
  include/libfreenect2/registration.h
  include/libfreenect2/resource.h
  include/libfreenect2/rgb_packet_processor.h
  include/libfreenect2/rgb_packet_stream_parser.h
  include/libfreenect2/threading.h
  
  src/transfer_pool.cpp
  src/event_loop.cpp
  src/usb_control.cpp
  src/double_buffer.cpp
  src/frame_listener_impl.cpp
  src/packet_pipeline.cpp
  src/rgb_packet_stream_parser.cpp
  src/rgb_packet_processor.cpp
  src/turbo_jpeg_rgb_packet_processor.cpp
  src/depth_packet_stream_parser.cpp
  src/depth_packet_processor.cpp
  src/cpu_depth_packet_processor.cpp
  src/resource.cpp
  src/command_transaction.cpp
  src/registration.cpp
  src/libfreenect2.cpp
  
  ${LIBFREENECT2_THREADING_SOURCE}
  ${RESOURCES_INC_FILE}
)

SET(LIBRARIES
  ${OpenCV_LIBS}
  ${OpenCV_LIBRARIES}
  ${LibUSB_LIBRARIES}
  ${TurboJPEG_LIBRARY}
  ${LIBFREENECT2_THREADING_LIBRARIES}
)

SET(RESOURCES
  11to16.bin 
  xTable.bin 
  zTable.bin
)

IF(ENABLE_OPENGL)
  auto_detect_lib(GLFW3)
  
  #Assuming local build witout global deps, ugly to to this automatic but to keep backward compatibiliy
  if(${GLFW3_FOUND} STREQUAL "FALSE")
    
    MESSAGE(STATUS "Building local GLFW")
    # GLFW
    SET(BUILD_SHARED_LIBS ON CACHE BOOL "Build shared libraries")
    SET(GLFW_BUILD_EXAMPLES OFF CACHE BOOL "Build the GLFW example programs")
    SET(GLFW_BUILD_TESTS OFF CACHE BOOL "Build the GLFW test programs")
    SET(GLFW_BUILD_DOCS OFF CACHE BOOL "Build the GLFW documentation")

    ADD_SUBDIRECTORY(${MY_DIR}/../../depends/glfw_src/ ${MY_DIR}/../../depends/glfw)
    INCLUDE_DIRECTORIES(${MY_DIR}/../../depends/glfw_src/include/)
    
    LIST(APPEND LIBRARIES
      glfw
    )
    
  else()
    INCLUDE_DIRECTORIES(${GLFW3_INCLUDE_DIRS})

    LINK_DIRECTORIES(${GLFW3_LIBRARY_DIRS})
    LIST(APPEND LIBRARIES
      ${GLFW3_LIBRARIES}
    )
  endif()
  
  SET(LIBFREENECT2_WITH_OPENGL_SUPPORT 1)
  LIST(APPEND SOURCES
    src/flextGL.c
    src/opengl_depth_packet_processor.cpp
  )
  
  LIST(APPEND RESOURCES
    src/shader/debug.fs
    src/shader/default.vs
    src/shader/filter1.fs
    src/shader/filter2.fs
    src/shader/stage1.fs
    src/shader/stage2.fs
  )
ENDIF(ENABLE_OPENGL)

IF(ENABLE_OPENCL)
  FIND_PACKAGE(OpenCL)

  IF(OPENCL_FOUND)      
    SET(LIBFREENECT2_WITH_OPENCL_SUPPORT 1)
    INCLUDE_DIRECTORIES(${OPENCL_INCLUDE_DIRS})
    
    LIST(APPEND SOURCES
      src/opencl_depth_packet_processor.cpp
    )

    LIST(APPEND LIBRARIES
      ${OPENCL_LIBRARIES}
    )
      
    LIST(APPEND RESOURCES
      src/opencl_depth_packet_processor.cl
    )
  ENDIF(OPENCL_FOUND)
ENDIF(ENABLE_OPENCL)

CONFIGURE_FILE("${MY_DIR}/include/libfreenect2/config.h.in" "${MY_DIR}/include/libfreenect2/config.h" @ONLY)
GENERATE_RESOURCES(${RESOURCES_INC_FILE} ${MY_DIR} ${RESOURCES})

ADD_DEFINITIONS(-DRESOURCES_INC)
ADD_LIBRARY(freenect2 SHARED ${SOURCES})
MESSAGE("Linking with these libraries: ${LIBRARIES}")
TARGET_LINK_LIBRARIES(freenect2 ${LIBRARIES})
  
ADD_EXECUTABLE(Protonect
  Protonect.cpp
)
  
TARGET_LINK_LIBRARIES(Protonect
  freenect2
)

CONFIGURE_FILE(freenect2.cmake.in "${PROJECT_BINARY_DIR}/freenect2Config.cmake" @ONLY)

INSTALL(TARGETS freenect2 DESTINATION lib)
INSTALL(DIRECTORY "${MY_DIR}/include/" DESTINATION include PATTERN "*.in" EXCLUDE)
IF(LIBFREENECT2_THREADING_TINYTHREAD)
  INSTALL(FILES  "${MY_DIR}/src/tinythread/tinythread.h" DESTINATION include/${PROJECT_NAME}/tinythread/)
ENDIF(LIBFREENECT2_THREADING_TINYTHREAD)
INSTALL(FILES "${PROJECT_BINARY_DIR}/freenect2Config.cmake" DESTINATION lib/cmake/freenect2/)
  
