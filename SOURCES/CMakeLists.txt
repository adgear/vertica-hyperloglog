cmake_minimum_required(VERSION 2.8)

# Make PROJECT_SOURCE_DIR, PROJECT_BINARY_DIR, and PROJECT_NAME available.
set(PROJECT_NAME vertica-udfs)
project(${PROJECT_NAME})

#####################
##     OPTIONS     ##
#####################

option(VEC_REPORT "Display compilation vectorization reports." OFF)
option(DEBUG_BUILD "Attach debug information to the executables." OFF)
option(BUILD_VERTICA_LIB "Build the .so library with UDFs for Vertica" OFF)
option(BUILD_HIVE_LIB "Build the .so library with Hive-friendly interface" OFF)
option(BUILD_VERTICA_TEST_DRIVER "Build a test program to show basic functionality of the underlying HLL algorithm" OFF)
option(BUILD_DATA_GEN "Build data generator for functional tests in Vertica" OFF)
option(BUILD_TESTS "Build all tests." OFF)
option(BUILD_BENCHMARK "Build benchmark to run HLL estimations." OFF)

#####################
##  COMPILE FLAGS  ##
#####################

set(CMAKE_CXX_FLAGS "-Wall -O3 -std=c++11 -Wno-unused-value")

set(VERTICA_INCLUDE ${SDK_HOME}/include)
include_directories(${VERTICA_INCLUDE} include src)

if(DEBUG_BUILD)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g")
endif()

if(VEC_REPORT)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fopt-info-vec-all=vecinfo.txt -ftree-vectorizer-verbose=7")
endif()

#####################
##  BUILD TARGETS  ##
#####################

if (BUILD_VERTICA_LIB)
  # Here we say where g++ should look for include files
  set(SDK_HOME /opt/vertica/sdk CACHE FILEPATH "Path to the Vertica SDK, by default /opt/vertica/sdk")
  if(NOT EXISTS ${SDK_HOME})
    message(FATAL_ERROR "Could not build. No SDK found at ${SDK_HOME} (maybe retry with -DSDK_HOME=<sdk_path>).")
  endif()

  # Here we add all source files to appear in libhll.so
  set(SOURCES src/bias_corrected_estimate.cpp src/linear_counting.cpp
  ${VERTICA_INCLUDE}/Vertica.h ${VERTICA_INCLUDE}/BuildInfo.h ${VERTICA_INCLUDE}/Vertica.cpp src/hll_vertica.cpp)
  add_library(hll SHARED ${SOURCES} src/HllCombine.cpp src/HllDistinctCount.cpp src/HllCreateSynopsis.cpp)
  add_library(loglogbeta SHARED ${SOURCES} src/LogLogBetaDistinctCount.cpp)
  add_library(hlldruid SHARED ${SOURCES} src/hll_druid.cpp src/HllDruidCombine.cpp src/HllDruidCreateSynopsis.cpp src/HllDruidDistinctCount.cpp src/murmur3_hash.cpp)
  set_target_properties(hll PROPERTIES COMPILE_FLAGS "${CMAKE_CXX_FLAGS} -march=native")
  set_target_properties(loglogbeta PROPERTIES COMPILE_FLAGS "${CMAKE_CXX_FLAGS} -march=native")
  set_target_properties(hlldruid PROPERTIES COMPILE_FLAGS "${CMAKE_CXX_FLAGS} -march=native")

  # Installation process just copies the binary HLL to Vertica lib folder
  set(LIB_DIR /opt/vertica/lib)
  install(
    TARGETS hll loglogbeta hlldruid
    DESTINATION ${LIB_DIR}
    PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE
  )
endif()

if (BUILD_HIVE_LIB)
  set(SOURCES src/bias_corrected_estimate.cpp src/linear_counting.cpp src/HllJava.cpp)
  # define HIVE_BUILD macro to compile the code without any call to glib's memcpy. This is to avoid troubles when using
  # HLL's code through JNA. We will make use of this macro in the following way:
  # #ifdef HIVE_BUILD
  #     for(uint32_t i=0; i<numberOfBuckets; ++i) {
  #         synopsis[i] = other.synopsis[i];
  #     }
  # #else
  #     memcpy(synopsis, other.synopsis, numberOfBuckets);
  # #endif
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DHIVE_BUILD")

  # we define this macro to use an older version of C++11 ABI
  # this is because we build first on newer machines and then copy the lib
  # to older systems, where the new C++ ABI is not available
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_GLIBCXX_USE_CXX11_ABI=0")
  add_library(hllhive SHARED ${SOURCES})
endif()

if (BUILD_VERTICA_TEST_DRIVER)
  add_executable(hll_driver tests/hll_driver.cpp src/bias_corrected_estimate.cpp src/linear_counting.cpp )
  add_executable(hll_druid_driver tests/hll_druid_driver.cpp src/hll_druid.cpp src/murmur3_hash.cpp)
endif()

if (BUILD_DATA_GEN)
  add_executable(data_gen tests/data_gen.cpp)
endif()

add_custom_target(check COMMAND ctest -V)
add_test(hll_test hll_test)

if (BUILD_TESTS)
  # This adds another subdirectory, which has 'project(gtest)'.
  # Thanks to EXCLUDE_FROM_ALL google test will not be installed
  # when running "make install"
  add_subdirectory(lib/googletest-1.7.0 EXCLUDE_FROM_ALL)

  enable_testing()
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DSOURCE_PATH='\"${CMAKE_CURRENT_LIST_DIR}\"'")
  # Include the gtest library. gtest_SOURCE_DIR is available due to
  # inclusion of googletest subdirectory with another cmake file
  include_directories(${gtest_SOURCE_DIR}/include ${gtest_SOURCE_DIR} src)

  # Linking to Hll.cpp. We can't link to libhll.so, because there would
  # be some symbols missing, e.g. Vertica::dummy()
  add_executable(hll_test tests/hll_druid_test.cpp tests/hll_test.cpp tests/hll_raw_test.cpp tests/bias_correction_test.cpp tests/linear_counting_test.cpp src/linear_counting.cpp src/bias_corrected_estimate.cpp src/hll_druid.cpp src/murmur3_hash.cpp)
  add_dependencies(check hll_test)
  # Standard linking to googletest stuff.
  target_link_libraries(hll_test gtest gtest_main)

  # Thanks to this one can run `make test' to run all the tests.
  # Every test to be run has to be added here
  # add_test(NAME that-test-I-made COMMAND runUnitTests)
endif()

if (BUILD_BENCHMARK)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DSOURCE_PATH='\"${CMAKE_CURRENT_LIST_DIR}\"'")
  add_executable(hll_benchmark tests/hll_benchmark.cpp src/linear_counting.cpp src/bias_corrected_estimate.cpp)
  add_dependencies(check hll_benchmark)
endif()
