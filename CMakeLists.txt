cmake_minimum_required(VERSION 2.8.8)

if(CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_CURRENT_BINARY_DIR AND NOT MSVC_IDE)
  message(FATAL_ERROR "In-source builds are not allowed.
Please create a directory and run cmake from there, passing the path to this source directory as the last argument.
This process created the file `CMakeCache.txt' and the directory `CMakeFiles'. Please delete them.")
endif()

project(OSRM C CXX)
set(OSRM_VERSION_MAJOR 4)
set(OSRM_VERSION_MINOR 8)
set(OSRM_VERSION_PATCH 1)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
include(CheckCXXCompilerFlag)
include(FindPackageHandleStandardArgs)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

set(bitness 32)
if(CMAKE_SIZEOF_VOID_P EQUAL 8)
  set(bitness 64)
  message(STATUS "Building on a 64 bit system")
else()
  message(WARNING "Building on a 32 bit system is unsupported")
endif()

if(WIN32 AND MSVC_VERSION LESS 1800)
  message(FATAL_ERROR "Building with Microsoft compiler needs Visual Studio 2013 or later (Express version works too)")
endif()

option(ENABLE_JSON_LOGGING "Adds additional JSON debug logging to the response" OFF)
option(DEBUG_GEOMETRY "Enables an option to dump GeoJSON of the final routing graph" OFF)
option(BUILD_TOOLS "Build OSRM tools" OFF)

include_directories(${CMAKE_CURRENT_SOURCE_DIR})
include_directories(${CMAKE_CURRENT_BINARY_DIR})
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include/)
include_directories(SYSTEM ${CMAKE_CURRENT_SOURCE_DIR}/third_party/)
include_directories(SYSTEM ${CMAKE_CURRENT_SOURCE_DIR}/third_party/libosmium/include/)

add_custom_target(FingerPrintConfigure ALL ${CMAKE_COMMAND}
  "-DOUTPUT_DIR=${CMAKE_CURRENT_BINARY_DIR}"
  "-DSOURCE_DIR=${CMAKE_CURRENT_SOURCE_DIR}"
  -P "${CMAKE_CURRENT_SOURCE_DIR}/cmake/FingerPrint-Config.cmake"
  COMMENT "Configuring revision fingerprint"
  VERBATIM)

add_custom_target(tests DEPENDS datastructure-tests algorithm-tests util-tests)
add_custom_target(benchmarks DEPENDS rtree-bench)

set(BOOST_COMPONENTS date_time filesystem iostreams program_options regex system thread unit_test_framework)

configure_file(
  ${CMAKE_CURRENT_SOURCE_DIR}/util/version.hpp.in
  ${CMAKE_CURRENT_BINARY_DIR}/util/version.hpp
)
file(GLOB ExtractorGlob extractor/*.cpp data_structures/hilbert_value.cpp)
file(GLOB ImporterGlob data_structures/import_edge.cpp data_structures/external_memory_node.cpp data_structures/raster_source.cpp)
add_library(IMPORT OBJECT ${ImporterGlob})
add_library(LOGGER OBJECT util/simple_logger.cpp)
add_library(PHANTOMNODE OBJECT data_structures/phantom_node.cpp)
add_library(RASTERSOURCE OBJECT data_structures/raster_source.cpp)
add_library(EXCEPTION OBJECT util/osrm_exception.cpp)
add_library(MERCATOR OBJECT util/mercator.cpp)
add_library(ANGLE OBJECT util/compute_angle.cpp)

set(ExtractorSources extract.cpp ${ExtractorGlob})
add_executable(osrm-extract ${ExtractorSources} $<TARGET_OBJECTS:COORDINATE> $<TARGET_OBJECTS:FINGERPRINT> $<TARGET_OBJECTS:IMPORT> $<TARGET_OBJECTS:LOGGER> $<TARGET_OBJECTS:EXCEPTION> $<TARGET_OBJECTS:MERCATOR> $<TARGET_OBJECTS:COMPRESSEDEDGE> $<TARGET_OBJECTS:GRAPHCOMPRESSOR> $<TARGET_OBJECTS:RESTRICTION> $<TARGET_OBJECTS:ANGLE>)

add_library(RESTRICTION OBJECT data_structures/restriction_map.cpp)
add_library(COMPRESSEDEDGE OBJECT data_structures/compressed_edge_container.cpp)
add_library(GRAPHCOMPRESSOR OBJECT algorithms/graph_compressor.cpp)

file(GLOB PrepareGlob contractor/*.cpp data_structures/hilbert_value.cpp {RestrictionMapGlob})
set(PrepareSources prepare.cpp ${PrepareGlob})
add_executable(osrm-prepare ${PrepareSources} $<TARGET_OBJECTS:ANGLE> $<TARGET_OBJECTS:FINGERPRINT> $<TARGET_OBJECTS:COORDINATE> $<TARGET_OBJECTS:IMPORT> $<TARGET_OBJECTS:LOGGER> $<TARGET_OBJECTS:RESTRICTION> $<TARGET_OBJECTS:EXCEPTION> $<TARGET_OBJECTS:MERCATOR> $<TARGET_OBJECTS:COMPRESSEDEDGE> $<TARGET_OBJECTS:GRAPHCOMPRESSOR>)

file(GLOB ServerGlob server/*.cpp)
file(GLOB DescriptorGlob descriptors/*.cpp)
file(GLOB DatastructureGlob data_structures/search_engine_data.cpp data_structures/route_parameters.cpp util/bearing.cpp)
file(GLOB CoordinateGlob data_structures/coordinate.cpp algorithms/coordinate_calculation.cpp)
file(GLOB AlgorithmGlob algorithms/polyline_compressor.cpp algorithms/polyline_formatter.cpp algorithms/douglas_peucker.cpp)
file(GLOB HttpGlob server/http/*.cpp)
file(GLOB LibOSRMGlob library/*.cpp)
file(GLOB DataStructureTestsGlob unit_tests/data_structures/*.cpp data_structures/hilbert_value.cpp)
file(GLOB AlgorithmTestsGlob unit_tests/algorithms/*.cpp algorithms/graph_compressor.cpp)
file(GLOB UtilTestsGlob unit_tests/util/*.cpp)

set(
  OSRMSources
  ${LibOSRMGlob}
  ${DescriptorGlob}
  ${DatastructureGlob}
  ${AlgorithmGlob}
  ${HttpGlob}
)

add_library(COORDINATE OBJECT ${CoordinateGlob})
add_library(OSRM ${OSRMSources} $<TARGET_OBJECTS:ANGLE> $<TARGET_OBJECTS:COORDINATE> $<TARGET_OBJECTS:FINGERPRINT> $<TARGET_OBJECTS:LOGGER> $<TARGET_OBJECTS:RESTRICTION> $<TARGET_OBJECTS:PHANTOMNODE> $<TARGET_OBJECTS:EXCEPTION> $<TARGET_OBJECTS:MERCATOR> $<TARGET_OBJECTS:IMPORT>)

add_library(FINGERPRINT OBJECT util/fingerprint.cpp)
add_dependencies(FINGERPRINT FingerPrintConfigure)
add_dependencies(OSRM FingerPrintConfigure)
set_target_properties(FINGERPRINT PROPERTIES LINKER_LANGUAGE CXX)

add_executable(osrm-routed routed.cpp ${ServerGlob} $<TARGET_OBJECTS:EXCEPTION>)
add_executable(osrm-datastore datastore.cpp $<TARGET_OBJECTS:COORDINATE> $<TARGET_OBJECTS:FINGERPRINT> $<TARGET_OBJECTS:LOGGER> $<TARGET_OBJECTS:EXCEPTION> $<TARGET_OBJECTS:MERCATOR>)

# Unit tests
add_executable(datastructure-tests EXCLUDE_FROM_ALL unit_tests/datastructure_tests.cpp ${DataStructureTestsGlob} $<TARGET_OBJECTS:COORDINATE> $<TARGET_OBJECTS:LOGGER> $<TARGET_OBJECTS:PHANTOMNODE> $<TARGET_OBJECTS:EXCEPTION> $<TARGET_OBJECTS:MERCATOR> $<TARGET_OBJECTS:COMPRESSEDEDGE> $<TARGET_OBJECTS:GRAPHCOMPRESSOR> $<TARGET_OBJECTS:RESTRICTION> $<TARGET_OBJECTS:RASTERSOURCE>)
add_executable(algorithm-tests EXCLUDE_FROM_ALL unit_tests/algorithm_tests.cpp ${AlgorithmTestsGlob} $<TARGET_OBJECTS:COORDINATE> $<TARGET_OBJECTS:LOGGER> $<TARGET_OBJECTS:PHANTOMNODE> $<TARGET_OBJECTS:EXCEPTION> $<TARGET_OBJECTS:RESTRICTION> $<TARGET_OBJECTS:COMPRESSEDEDGE>)
add_executable(util-tests EXCLUDE_FROM_ALL unit_tests/util_tests.cpp ${UtilTestsGlob})

# Benchmarks
add_executable(rtree-bench EXCLUDE_FROM_ALL benchmarks/static_rtree.cpp $<TARGET_OBJECTS:COORDINATE> $<TARGET_OBJECTS:LOGGER> $<TARGET_OBJECTS:PHANTOMNODE> $<TARGET_OBJECTS:EXCEPTION> $<TARGET_OBJECTS:MERCATOR>)

# Check the release mode
if(NOT CMAKE_BUILD_TYPE MATCHES Debug)
  set(CMAKE_BUILD_TYPE Release)
endif()
if(CMAKE_BUILD_TYPE MATCHES Debug)
  message(STATUS "Configuring OSRM in debug mode")
  if(NOT ${CMAKE_CXX_COMPILER_ID} STREQUAL "MSVC")

  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-inline -fno-omit-frame-pointer")

  if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Og -ggdb")
  endif()

  endif()
endif()
if(CMAKE_BUILD_TYPE MATCHES Release)
  message(STATUS "Configuring OSRM in release mode")
  # Check if LTO is available
  check_cxx_compiler_flag("-flto" LTO_AVAILABLE)
  if(LTO_AVAILABLE)
    set(OLD_CXX_FLAGS ${CMAKE_CXX_FLAGS})
    # GCC in addition allows parallelizing LTO
    if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
      include(ProcessorCount)
      ProcessorCount(NPROC)
      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -flto=${NPROC}")
    else()
      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -flto")
    endif()
    set(CHECK_LTO_SRC "int main(){return 0;}")
    check_cxx_source_compiles("${CHECK_LTO_SRC}" LTO_WORKS)
    if(LTO_WORKS)
      message(STATUS "LTO working")
    else()
      message(STATUS "LTO broken")
      set(CMAKE_CXX_FLAGS "${OLD_CXX_FLAGS}")
    endif()

    # Since gcc 4.9 the LTO format is non-standart ('slim'), so we need to use the build-in tools
    if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU" AND
        NOT "${CMAKE_CXX_COMPILER_VERSION}" VERSION_LESS "4.9.0" AND NOT MINGW)
      message(STATUS "Using gcc specific binutils for LTO.")
      set(CMAKE_AR     "/usr/bin/gcc-ar")
      set(CMAKE_RANLIB "/usr/bin/gcc-ranlib")
    endif()

    if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU" AND "${CMAKE_CXX_COMPILER_VERSION}" VERSION_LESS "4.9.0")
      message(STATUS "Disabling LTO on GCC < 4.9.0 since it is broken, see: https://gcc.gnu.org/bugzilla/show_bug.cgi?id=57038")
      set(CMAKE_CXX_FLAGS "${OLD_CXX_FLAGS}")
    endif()
  endif()
endif()

if(NOT WIN32)
  add_definitions(-DBOOST_TEST_DYN_LINK)
endif()

# Configuring compilers
if(${CMAKE_CXX_COMPILER_ID} STREQUAL "Clang")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -pedantic -Wuninitialized -Wunreachable-code -Wstrict-overflow=2 -D_FORTIFY_SOURCE=2 -fPIC")
elseif(${CMAKE_CXX_COMPILER_ID} STREQUAL "GNU")
  set(COLOR_FLAG "-fdiagnostics-color=auto")
  check_cxx_compiler_flag("-fdiagnostics-color=auto" HAS_COLOR_FLAG)
  if(NOT HAS_COLOR_FLAG)
    set(COLOR_FLAG "")
  endif()
  # using GCC
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -pedantic -Wuninitialized -Wunreachable-code -Wstrict-overflow=1 -D_FORTIFY_SOURCE=2 ${COLOR_FLAG} -fPIC")
  if(WIN32) # using mingw
    add_definitions(-D_USE_MATH_DEFINES) # define M_PI, M_1_PI etc.
    add_definitions(-DWIN32)
    set(OPTIONAL_SOCKET_LIBS ws2_32 wsock32)
  endif()
elseif(${CMAKE_CXX_COMPILER_ID} STREQUAL "Intel")
  # using Intel C++
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -static-intel -wd10237 -Wall -ipo -fPIC")
elseif(${CMAKE_CXX_COMPILER_ID} STREQUAL "MSVC")
  # using Visual Studio C++
  set(BOOST_COMPONENTS ${BOOST_COMPONENTS} date_time chrono zlib)
  add_definitions(-D_CRT_SECURE_NO_WARNINGS)
  add_definitions(-DNOMINMAX) # avoid min and max macros that can break compilation
  add_definitions(-D_USE_MATH_DEFINES) # define M_PI
  add_definitions(-D_WIN32_WINNT=0x0501)
  add_definitions(-DXML_STATIC)
  find_library(ws2_32_LIBRARY_PATH ws2_32)
  target_link_libraries(osrm-extract wsock32 ws2_32)
endif()

# Configuring linker
execute_process(COMMAND ${CMAKE_CXX_COMPILER} "-Wl,--version" ERROR_QUIET OUTPUT_VARIABLE LINKER_VERSION)
# For ld.gold and ld.bfs (the GNU linkers) we optimize hard
if("${LINKER_VERSION}" MATCHES "GNU gold" OR "${LINKER_VERSION}" MATCHES "GNU ld")
  message(STATUS "Setting linker optimizations")
  if(NOT ${CMAKE_CXX_COMPILER_ID} STREQUAL "MSVC")
    # Tell compiler to put every function in separate section, linker can then match sections and functions
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -ffunction-sections -fdata-sections")
    # Tell linker to do dead code and data eminination during link time discarding sections
    set(LINKER_FLAGS "${LINKER_FLAGS} -Wl,--gc-sections")
  endif()
  # Default linker optimization flags
  set(LINKER_FLAGS "${LINKER_FLAGS} -Wl,-O1 -Wl,--hash-style=gnu -Wl,--sort-common")
else()
  message(STATUS "Using unknown linker, not setting linker optimizations")
endif ()
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${LINKER_FLAGS}")
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} ${LINKER_FLAGS}")
set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} ${LINKER_FLAGS}")

# Activate C++11
if(NOT ${CMAKE_CXX_COMPILER_ID} STREQUAL "MSVC")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 ")
endif()

# Configuring other platform dependencies
if(APPLE)
  set(CMAKE_OSX_ARCHITECTURES "x86_64")
  message(STATUS "Set Architecture to x64 on OS X")
  exec_program(uname ARGS -v  OUTPUT_VARIABLE DARWIN_VERSION)
  string(REGEX MATCH "[0-9]+" DARWIN_VERSION ${DARWIN_VERSION})
  if(OSXLIBSTD)
    message(STATUS "linking against ${OSXLIBSTD}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=${OSXLIBSTD}")
  elseif(DARWIN_VERSION GREATER 12)
    message(STATUS "linking against libc++")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")
  endif()
endif()

if(UNIX AND NOT APPLE)
  target_link_libraries(osrm-prepare rt)
  target_link_libraries(osrm-datastore rt)
  target_link_libraries(OSRM rt)
endif()

#Check Boost
find_package(Boost 1.49.0 COMPONENTS ${BOOST_COMPONENTS} REQUIRED)
if(NOT Boost_FOUND)
  message(FATAL_ERROR "Fatal error: Boost (version >= 1.49.0) required.\n")
endif()
include_directories(SYSTEM ${Boost_INCLUDE_DIRS})

target_link_libraries(OSRM ${Boost_LIBRARIES})
target_link_libraries(osrm-extract ${Boost_LIBRARIES})
target_link_libraries(osrm-prepare ${Boost_LIBRARIES})
target_link_libraries(osrm-routed ${Boost_LIBRARIES} ${OPTIONAL_SOCKET_LIBS} OSRM)
target_link_libraries(osrm-datastore ${Boost_LIBRARIES})
target_link_libraries(datastructure-tests ${Boost_LIBRARIES})
target_link_libraries(algorithm-tests ${Boost_LIBRARIES} ${OPTIONAL_SOCKET_LIBS} OSRM)
target_link_libraries(util-tests ${Boost_LIBRARIES})
target_link_libraries(rtree-bench ${Boost_LIBRARIES})

find_package(Threads REQUIRED)
target_link_libraries(osrm-extract ${CMAKE_THREAD_LIBS_INIT})
target_link_libraries(osrm-datastore ${CMAKE_THREAD_LIBS_INIT})
target_link_libraries(osrm-prepare ${CMAKE_THREAD_LIBS_INIT})
target_link_libraries(OSRM ${CMAKE_THREAD_LIBS_INIT})
target_link_libraries(datastructure-tests ${CMAKE_THREAD_LIBS_INIT})
target_link_libraries(algorithm-tests ${CMAKE_THREAD_LIBS_INIT})
target_link_libraries(rtree-bench ${CMAKE_THREAD_LIBS_INIT})

find_package(TBB REQUIRED)
if(WIN32 AND CMAKE_BUILD_TYPE MATCHES Debug)
  set(TBB_LIBRARIES ${TBB_DEBUG_LIBRARIES})
endif()
target_link_libraries(osrm-datastore ${TBB_LIBRARIES})
target_link_libraries(osrm-extract ${TBB_LIBRARIES})
target_link_libraries(osrm-prepare ${TBB_LIBRARIES})
target_link_libraries(osrm-routed ${TBB_LIBRARIES})
target_link_libraries(datastructure-tests ${TBB_LIBRARIES})
target_link_libraries(algorithm-tests ${TBB_LIBRARIES})
target_link_libraries(rtree-bench ${TBB_LIBRARIES})
include_directories(SYSTEM ${TBB_INCLUDE_DIR})

find_package( Luabind REQUIRED )
include(check_luabind)

include_directories(SYSTEM ${LUABIND_INCLUDE_DIR})
target_link_libraries(osrm-extract ${LUABIND_LIBRARY})
target_link_libraries(osrm-prepare ${LUABIND_LIBRARY})

if(LUAJIT_FOUND)
  target_link_libraries(osrm-extract ${LUAJIT_LIBRARIES})
  target_link_libraries(osrm-prepare ${LUAJIT_LIBRARIES})
else()
  target_link_libraries(osrm-extract ${LUA_LIBRARY})
  target_link_libraries(osrm-prepare ${LUA_LIBRARY})
endif()
include_directories(SYSTEM ${LUA_INCLUDE_DIR})

find_package(EXPAT REQUIRED)
include_directories(SYSTEM ${EXPAT_INCLUDE_DIRS})
target_link_libraries(osrm-extract ${EXPAT_LIBRARIES})

find_package(STXXL REQUIRED)
include_directories(SYSTEM ${STXXL_INCLUDE_DIR})
target_link_libraries(OSRM ${STXXL_LIBRARY})
target_link_libraries(osrm-extract ${STXXL_LIBRARY})
target_link_libraries(osrm-prepare ${STXXL_LIBRARY})

set(OpenMP_FIND_QUIETLY ON)
find_package(OpenMP)
if(OPENMP_FOUND)
  message(STATUS "OpenMP support found. Linking just in case for stxxl")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
endif()

find_package(BZip2 REQUIRED)
include_directories(SYSTEM ${BZIP_INCLUDE_DIRS})
target_link_libraries(osrm-extract ${BZIP2_LIBRARIES})

find_package(ZLIB REQUIRED)
include_directories(SYSTEM ${ZLIB_INCLUDE_DIRS})
target_link_libraries(osrm-extract ${ZLIB_LIBRARY})
target_link_libraries(osrm-routed ${ZLIB_LIBRARY})

if (ENABLE_JSON_LOGGING)
  message(STATUS "Enabling json logging")
  add_definitions(-DENABLE_JSON_LOGGING)
endif()

if (DEBUG_GEOMETRY)
  message(STATUS "Enabling final edge weight GeoJSON output option")
  add_definitions(-DDEBUG_GEOMETRY)
endif()

if(BUILD_TOOLS)
  message(STATUS "Activating OSRM internal tools")
  find_package(GDAL)
  if(GDAL_FOUND)
    add_executable(osrm-components tools/components.cpp $<TARGET_OBJECTS:FINGERPRINT> $<TARGET_OBJECTS:IMPORT> $<TARGET_OBJECTS:COORDINATE> $<TARGET_OBJECTS:LOGGER> $<TARGET_OBJECTS:RESTRICTION> $<TARGET_OBJECTS:EXCEPTION> $<TARGET_OBJECTS:MERCATOR>)
    target_link_libraries(osrm-components ${TBB_LIBRARIES})
    include_directories(SYSTEM ${GDAL_INCLUDE_DIR})
    target_link_libraries(osrm-components ${GDAL_LIBRARIES} ${Boost_LIBRARIES})
    install(TARGETS osrm-components DESTINATION bin)
  else()
    message(FATAL_ERROR "libgdal and/or development headers not found")
  endif()
  add_executable(osrm-cli tools/simpleclient.cpp $<TARGET_OBJECTS:EXCEPTION> $<TARGET_OBJECTS:LOGGER> $<TARGET_OBJECTS:COORDINATE>)
  target_link_libraries(osrm-cli ${Boost_LIBRARIES} ${OPTIONAL_SOCKET_LIBS} OSRM)
  target_link_libraries(osrm-cli ${TBB_LIBRARIES})
  add_executable(osrm-io-benchmark tools/io-benchmark.cpp $<TARGET_OBJECTS:EXCEPTION> $<TARGET_OBJECTS:LOGGER>)
  target_link_libraries(osrm-io-benchmark ${Boost_LIBRARIES})
  add_executable(osrm-unlock-all tools/unlock_all_mutexes.cpp $<TARGET_OBJECTS:LOGGER> $<TARGET_OBJECTS:EXCEPTION>)
  target_link_libraries(osrm-unlock-all ${Boost_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT})
  if(UNIX AND NOT APPLE)
    target_link_libraries(osrm-unlock-all rt)
  endif()
  add_executable(osrm-check-hsgr tools/check-hsgr.cpp $<TARGET_OBJECTS:FINGERPRINT> $<TARGET_OBJECTS:EXCEPTION> $<TARGET_OBJECTS:LOGGER> $<TARGET_OBJECTS:IMPORT>)
  target_link_libraries(osrm-check-hsgr ${Boost_LIBRARIES} ${TBB_LIBRARIES})
  add_executable(osrm-springclean tools/springclean.cpp $<TARGET_OBJECTS:FINGERPRINT> $<TARGET_OBJECTS:LOGGER> $<TARGET_OBJECTS:EXCEPTION>)
  target_link_libraries(osrm-springclean ${Boost_LIBRARIES})

  install(TARGETS osrm-cli DESTINATION bin)
  install(TARGETS osrm-io-benchmark DESTINATION bin)
  install(TARGETS osrm-unlock-all DESTINATION bin)
  install(TARGETS osrm-check-hsgr DESTINATION bin)
  install(TARGETS osrm-springclean DESTINATION bin)
endif()

file(GLOB InstallGlob include/osrm/*.hpp library/osrm.hpp)
file(GLOB VariantGlob third_party/variant/*.hpp)

# Add RPATH info to executables so that when they are run after being installed
# (i.e., from /usr/local/bin/) the linker can find library dependencies. For
# more info see http://www.cmake.org/Wiki/CMake_RPATH_handling
set_property(TARGET osrm-extract PROPERTY INSTALL_RPATH_USE_LINK_PATH TRUE)
set_property(TARGET osrm-prepare PROPERTY INSTALL_RPATH_USE_LINK_PATH TRUE)
set_property(TARGET osrm-datastore PROPERTY INSTALL_RPATH_USE_LINK_PATH TRUE)
set_property(TARGET osrm-routed PROPERTY INSTALL_RPATH_USE_LINK_PATH TRUE)

install(FILES ${InstallGlob} DESTINATION include/osrm)
install(FILES ${VariantGlob} DESTINATION include/variant)
install(TARGETS osrm-extract DESTINATION bin)
install(TARGETS osrm-prepare DESTINATION bin)
install(TARGETS osrm-datastore DESTINATION bin)
install(TARGETS osrm-routed DESTINATION bin)
install(TARGETS OSRM DESTINATION lib)
list(GET Boost_LIBRARIES 1 BOOST_LIBRARY_FIRST)
get_filename_component(BOOST_LIBRARY_LISTING "${BOOST_LIBRARY_FIRST}" PATH)
set(BOOST_LIBRARY_LISTING "-L${BOOST_LIBRARY_LISTING}")
foreach(lib ${Boost_LIBRARIES})
  get_filename_component(BOOST_LIBRARY_NAME "${lib}" NAME_WE)
  string(REPLACE "lib" "" BOOST_LIBRARY_NAME ${BOOST_LIBRARY_NAME})
  set(BOOST_LIBRARY_LISTING "${BOOST_LIBRARY_LISTING} -l${BOOST_LIBRARY_NAME}")
endforeach()

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/cmake/pkgconfig.in libosrm.pc @ONLY)
install(FILES ${PROJECT_BINARY_DIR}/libosrm.pc DESTINATION lib/pkgconfig)

if(BUILD_DEBIAN_PACKAGE)
  include(CPackDebianConfig)
  include(CPack)
endif()

# add a target to generate API documentation with Doxygen
find_package(Doxygen)
if(DOXYGEN_FOUND)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/Doxyfile.in ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile @ONLY)
add_custom_target(doc
${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile
WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
COMMENT "Generating API documentation with Doxygen" VERBATIM
)
endif()

# prefix compilation with ccache by default if available and on clang or gcc
if(${CMAKE_CXX_COMPILER_ID} STREQUAL "Clang" OR ${CMAKE_CXX_COMPILER_ID} STREQUAL "GNU")
  find_program(CCACHE_FOUND ccache)
  if(CCACHE_FOUND)
    set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ccache)
    set_property(GLOBAL PROPERTY RULE_LAUNCH_LINK ccache)
    set(ENV{CCACHE_CPP2} "true")
  endif()
endif()
