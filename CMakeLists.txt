cmake_minimum_required(VERSION 3.16)
project(ModeratedLMS VERSION 1.0.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

option(LMS_BUILD_TESTS "Build the GoogleTest suite" ON)
option(LMS_COVERAGE "Instrument for gcov line coverage (GCC/Clang)" OFF)
option(LMS_WARNINGS_AS_ERRORS "Treat compiler warnings as errors" OFF)

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

# ---- Dependencies ---------------------------------------------------------
find_package(SQLite3 REQUIRED)
find_package(PkgConfig REQUIRED)
pkg_check_modules(SODIUM REQUIRED IMPORTED_TARGET libsodium)

# ---- Embed the SQL schema so the code can never drift from schema/schema.sql
file(READ "${CMAKE_SOURCE_DIR}/schema/schema.sql" LMS_SCHEMA_SQL)
set_property(DIRECTORY APPEND PROPERTY CMAKE_CONFIGURE_DEPENDS "${CMAKE_SOURCE_DIR}/schema/schema.sql")
configure_file("${CMAKE_SOURCE_DIR}/src/data/sqlite/SchemaSql.cpp.in"
               "${CMAKE_BINARY_DIR}/generated/SchemaSql.cpp" @ONLY)

# ---- Core library (everything except main) --------------------------------
file(GLOB_RECURSE LMS_SOURCES CONFIGURE_DEPENDS "${CMAKE_SOURCE_DIR}/src/*.cpp")
list(FILTER LMS_SOURCES EXCLUDE REGEX ".*/src/main\\.cpp$")
add_library(lms_core STATIC ${LMS_SOURCES} "${CMAKE_BINARY_DIR}/generated/SchemaSql.cpp")
target_include_directories(lms_core PUBLIC "${CMAKE_SOURCE_DIR}/src")
target_link_libraries(lms_core PUBLIC SQLite::SQLite3 PkgConfig::SODIUM)

if(MSVC)
  target_compile_options(lms_core PUBLIC /W4)
else()
  target_compile_options(lms_core PUBLIC -Wall -Wextra -Wpedantic -Wshadow)
  if(LMS_WARNINGS_AS_ERRORS)
    target_compile_options(lms_core PUBLIC -Werror)
  endif()
endif()

if(LMS_COVERAGE AND NOT MSVC)
  target_compile_options(lms_core PUBLIC --coverage -O0 -g)
  target_link_libraries(lms_core PUBLIC --coverage)
endif()

# ---- Application ----------------------------------------------------------
add_executable(lms "${CMAKE_SOURCE_DIR}/src/main.cpp")
target_link_libraries(lms PRIVATE lms_core)

# ---- Tests ----------------------------------------------------------------
if(LMS_BUILD_TESTS)
  enable_testing()
  find_package(GTest QUIET)
  if(NOT GTest_FOUND)
    include(FetchContent)
    FetchContent_Declare(googletest
      URL https://github.com/google/googletest/archive/refs/tags/v1.14.0.zip)
    set(INSTALL_GTEST OFF CACHE BOOL "" FORCE)
    FetchContent_MakeAvailable(googletest)
  endif()
  file(GLOB LMS_TEST_SOURCES CONFIGURE_DEPENDS "${CMAKE_SOURCE_DIR}/tests/*.cpp")
  add_executable(lms_tests ${LMS_TEST_SOURCES})
  target_link_libraries(lms_tests PRIVATE lms_core GTest::gtest GTest::gtest_main)
  include(GoogleTest)
  gtest_discover_tests(lms_tests DISCOVERY_MODE PRE_TEST)
endif()
