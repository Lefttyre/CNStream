# ----------------------------------------------------------------------------
#  Root CMake file for CNStream
# ----------------------------------------------------------------------------

# Disable in-source builds to prevent source tree corruption.
if(" ${CMAKE_SOURCE_DIR}" STREQUAL " ${CMAKE_BINARY_DIR}")
  message(FATAL_ERROR "Please create a separate directory for build files.")
endif()

cmake_minimum_required(VERSION 2.8.7)
if(POLICY CMP0046)
  cmake_policy(SET CMP0046 NEW)
endif()
if(POLICY CMP0054)
  cmake_policy(SET CMP0054 NEW)
endif()

# ---[ CNStream project
project(CNStream C CXX)

if(CMAKE_TOOLCHAIN_FILE) 
  message("cross compile enabled, CMAKE_TOOLCHAIN_FILE: ${CMAKE_TOOLCHAIN_FILE}")
endif()

# ---[ Options
option(build_display   "build module display" ON)
option(build_encode    "build module encode" ON)
option(build_fps_stats "build module fps_stats" ON)
option(build_inference "build module inference" ON)
option(build_osd       "build module osd" ON)
option(build_source    "build module source" ON)
option(build_track     "build module track" ON)
option(build_discard_frame "build module discard_frame" ON)
option(cnstream_build_integrated_lib "build modules into one library" ON)
option(cnstream_build_tests "build all of modules' unit test" ON)

option(cnstream_build_samples "build sample programs" ON)
option(cnstream_build_apps "build apps" ON)
option(cnstream_build_test_coverage  "Test code coverage" OFF)

option(RELEASE "build type" ON)
option(WITH_FFMPEG "with ffmpeg" ON)
option(WITH_OPENCV "with opencv" ON)
option(WITH_CHINESE "with chinese" OFF)
option(WITH_RTSP "with rtsp" ON)
include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
CHECK_CXX_COMPILER_FLAG("-std=c++0x" COMPILER_SUPPORTS_CXX0X)
if(COMPILER_SUPPORTS_CXX11)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wl,--as-needed -D_REENTRANT")
elseif(COMPILER_SUPPORTS_CXX0X)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x -Wl,--as-needed -D_REENTRANT")
else()
  message(STATUS "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support. Please use a different C++ compiler.")
endif()

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -D_REENTRANT")

#######################################################################
if(MLU STREQUAL MLU100)
  message("generate MLU100 Makefile")
  set(MLU_PLATFORM MLU100)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DCNS_MLU100")
elseif(MLU STREQUAL MLU270)
  message("generate MLU270 Makefile")
  set(MLU_PLATFORM MLU270)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DCNS_MLU270")
else()
  message("generate default(MLU100) Makefile")
  set(MLU_PLATFORM MLU100)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DCNS_MLU100")
endif()

# -- Build Flags
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC -Wall -Werror")
if(cnstream_build_test_coverage)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fprofile-arcs -ftest-coverage")
endif()
if(RELEASE)
  set(CMAKE_BUILD_TYPE Release)
  set(CMAKE_CXX_FLAGS_RELEASE "-DNDEBUG -O2 -s")
else()
  set(CMAKE_BUILD_TYPE Debug)
  set(CMAKE_CXX_FLAGS_DEBUG "-DDEBUG -g")
endif()

if(USE_libstdcpp)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libstdc++")
  message("-- Warning: forcing libstdc++ (controlled by USE_libstdcpp option in cmake)")
endif()

# ---[ Google-gflags
include("${PROJECT_SOURCE_DIR}/cmake/FindGFlags.cmake")

list(APPEND 3RDPARTY_INCLUDE_DIRS ${GFLAGS_INCLUDE_DIRS})
list(APPEND 3RDPARTY_LIBS ${GFLAGS_LIBRARIES})

# ---[ Google-glog
include("${PROJECT_SOURCE_DIR}/cmake/FindGlog.cmake")

list(APPEND 3RDPARTY_INCLUDE_DIRS ${GLOG_INCLUDE_DIRS})
list(APPEND 3RDPARTY_LIBS ${GLOG_LIBRARIES})

##OpenCV
if(WITH_OPENCV)
  set(OpenCV_FIND_QUIETLY true)
  #find_package(OpenCV REQUIRED) //this will include opencv_ts which result in crash on centos
  find_package(OpenCV OPTIONAL_COMPONENTS imgcodecs)
  set(imgcodecs_libs ${OpenCV_LIBS})
  find_package(OpenCV REQUIRED core imgproc highgui features2d)
  if(OpenCV_FOUND)
    message(STATUS "OpenCV Found")
    set(HAVE_OPENCV true)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DHAVE_OPENCV")
  endif()
  if(imgcodecs_FOUND)
    list(APPEND OpenCV_LIBS imgcodecs_libs)
  endif()
else()
  set (HAVE_OPENCV false)
endif()

##FFmpeg
if(WITH_FFMPEG)
  if (MLU_PLATFORM STREQUAL MLU220_SOC)
    set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)
    include_directories(${PROJECT_SOURCE_DIR}/3rdparty/aarch64/ffmpeg/include)
    set(FFMPEG_LIBRARIES ${PROJECT_SOURCE_DIR}/3rdparty/aarch64/ffmpeg/lib/libavcodec.so)
    list(APPEND FFMPEG_LIBRARIES ${PROJECT_SOURCE_DIR}/3rdparty/aarch64/ffmpeg/lib/libavdevice.so)
    list(APPEND FFMPEG_LIBRARIES ${PROJECT_SOURCE_DIR}/3rdparty/aarch64/ffmpeg/lib/libavfilter.so)
    list(APPEND FFMPEG_LIBRARIES ${PROJECT_SOURCE_DIR}/3rdparty/aarch64/ffmpeg/lib/libavformat.so)
    list(APPEND FFMPEG_LIBRARIES ${PROJECT_SOURCE_DIR}/3rdparty/aarch64/ffmpeg/lib/libavutil.so)
    list(APPEND FFMPEG_LIBRARIES ${PROJECT_SOURCE_DIR}/3rdparty/aarch64/ffmpeg/lib/libswresample.so)
    list(APPEND FFMPEG_LIBRARIES ${PROJECT_SOURCE_DIR}/3rdparty/aarch64/ffmpeg/lib/libswscale.so)
    set(HAVE_FFMPEG true)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DHAVE_FFMPEG")
  else()
   set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake) 
   find_package(FFmpeg REQUIRED)
   if (FFMPEG_FOUND)
     include_directories(${FFMPEG_INCLUDE_DIR})
     message(STATUS "ffmpeg include: ${FFMPEG_INCLUDE_DIR}")
     message(STATUS "ffmpeg libraries: ${FFMPEG_LIBRARIES}")
     set(HAVE_FFMPEG true)
     set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DHAVE_FFMPEG")
     message(STATUS "HAVE_FFMPEG enabled")
   else (FFMPEG_FOUND)
     message(FATAL_ERROR "Can't find FFmpeg libs!")
     set (HAVE_FFMPEG false)
   endif (FFMPEG_FOUND)
  endif()
else()
  set (HAVE_FFMPEG false)
endif()

##Chinses label
if(WITH_CHINESE)
  find_package(Freetype REQUIRED)
  if (FREETYPE_FOUND)
    include_directories(${FREETYPE_INCLUDE_DIR_ft2build})
    message(STATUS "freetype include: ${FREETYPE_INCLUDE_DIR_ft2build}")
    message(STATUS "freetype libraries: ${FREETYPE_LIBRARIES}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DHAVE_FREETYPE")
  else (FREETYPE_FOUND)
    message(FATAL_ERROR "Can't find Freetype libs!")
  endif (FREETYPE_FOUND)
endif()

# ---[ dependencies
# neuware include/libs
if(NOT DEFINED ENV{NEUWARE_HOME})
  set(ENV{NEUWARE_HOME} /usr/local/neuware)
endif()
if(EXISTS ${PROJECT_SOURCE_DIR}/mlu/${MLU_PLATFORM})
  message(STATUS "Local MLU libs found")
  #cp neuware/.h or .so for cnstream
  if(MLU_PLATFORM STREQUAL MLU100)
    execute_process(COMMAND sh ${PROJECT_SOURCE_DIR}/tools/copy_neuware_mlu100.sh $ENV{NEUWARE_HOME} ${PROJECT_SOURCE_DIR}/mlu/${MLU_PLATFORM}/include  ${PROJECT_SOURCE_DIR}/mlu/${MLU_PLATFORM}/libs/${CMAKE_SYSTEM_PROCESSOR})
    message(STATUS "MLU100 neuware files copy Done")
    list(APPEND CN_LIBS cnstream-toolkit cncodec cnrt)
  elseif(MLU_PLATFORM STREQUAL MLU270)
    execute_process(COMMAND sh ${PROJECT_SOURCE_DIR}/tools/copy_neuware_mlu270.sh $ENV{NEUWARE_HOME} ${PROJECT_SOURCE_DIR}/mlu/${MLU_PLATFORM}/include  ${PROJECT_SOURCE_DIR}/mlu/${MLU_PLATFORM}/libs/${CMAKE_SYSTEM_PROCESSOR})
    message(STATUS "MLU270 neuware files copy Done")
    list(APPEND CN_LIBS cnstream-toolkit cncodec cnrt)
  else()
    message(FATAL_ERROR "MLU_PLATFORM not supported!")
  endif()
  include_directories(${PROJECT_SOURCE_DIR}/mlu/${MLU_PLATFORM}/include)
  link_directories(${PROJECT_SOURCE_DIR}/mlu/${MLU_PLATFORM}/libs/${CMAKE_SYSTEM_PROCESSOR})
else()
  include_directories("$ENV{NEUWARE_HOME}/include")
  link_directories("$ENV{NEUWARE_HOME}/lib64")
  list(APPEND CN_LIBS cnstream-toolkit cncodec cnrt)
endif()

# cns modules libs
link_directories(${PROJECT_SOURCE_DIR}/lib)


# ---[ Includes
include_directories(${PROJECT_SOURCE_DIR}/modules/core/include)

# ---[ Rapidjson
include_directories(${PROJECT_SOURCE_DIR}/3rdparty/rapidjson/include)

if(cnstream_build_tests)
  add_definitions(-DUNIT_TEST)
  # This must be set in the root directory for the tests to be run by
  # 'make test' or ctest.
  enable_testing()
endif()

# ---[ Subdirectories
add_subdirectory(3rdparty)
add_subdirectory(modules)
add_subdirectory(tools)
if(cnstream_build_samples)
  add_subdirectory(samples)
endif()

if(EXISTS ${PROJECT_SOURCE_DIR}/apps)
  if(cnstream_build_apps)
    message(STATUS "generate apps' makefile")
    add_subdirectory(apps)
  endif()
endif()
