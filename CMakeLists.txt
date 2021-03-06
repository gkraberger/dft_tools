# Default to Release build type
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Type of build" FORCE)
endif()
message( STATUS "-------- BUILD-TYPE: ${CMAKE_BUILD_TYPE} -------------")

# start configuration
cmake_minimum_required(VERSION 2.8)
project(dft_tools C CXX Fortran)

# Use shared libraries
set(BUILD_SHARED_LIBS ON)

# Load TRIQS and Cpp2Py
find_package(TRIQS 2.0 REQUIRED)
find_package(Cpp2Py 1.5 REQUIRED)

if (NOT ${TRIQS_WITH_PYTHON_SUPPORT})
 MESSAGE(FATAL_ERROR "dft_tools require Python support in TRIQS")
endif()

# Default Install directory to TRIQS_ROOT if not given. Checks an absolute name is given.
if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT OR (NOT IS_ABSOLUTE ${CMAKE_INSTALL_PREFIX}))
  message(STATUS " No install prefix given (or invalid). Defaulting to TRIQS_ROOT")
  set(CMAKE_INSTALL_PREFIX ${TRIQS_ROOT} CACHE PATH "default install path" FORCE)
endif()
message(STATUS "-------- CMAKE_INSTALL_PREFIX: ${CMAKE_INSTALL_PREFIX} -------------")

# Use git to detect Cthyb version and git hash
execute_process(COMMAND bash "-c" "${TRIQS_GIT_EXECUTABLE} describe | sed \"s/-\\([0-9]*\\)-.*/.\\1/g\""
 WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}"
 OUTPUT_VARIABLE   DFT_TOOLS_VERSION
 OUTPUT_STRIP_TRAILING_WHITESPACE
)
if(NOT DFT_TOOLS_VERSION)
  set(DFT_TOOLS_VERSION 1.5)
endif()
triqs_get_git_hash_of_source_dir(DFT_TOOLS_GIT_HASH)
MESSAGE(STATUS "Dft_tools version : ${DFT_TOOLS_VERSION}")
message(STATUS "Git hash: ${DFT_TOOLS_GIT_HASH}")

add_subdirectory(fortran/dmftproj) 

# Add the compiling options (-D... ) for C++
message(STATUS "TRIQS : Adding compilation flags detected by the library (C++11/14, libc++, etc...) ")

add_subdirectory(c++)
add_subdirectory(python python/triqs_dft_tools)
add_subdirectory(shells)

#------------------------
# tests
#------------------------

option(TEST_COVERAGE "Analyze the coverage of tests" OFF)

# perform tests with coverage info
if (${TEST_COVERAGE})
  # we try to locate the coverage program
  find_program(PYTHON_COVERAGE python-coverage)
  find_program(PYTHON_COVERAGE coverage)
  if(NOT PYTHON_COVERAGE)
      message(FATAL_ERROR "Program coverage (or python-coverage) not found.\nEither set PYTHON_COVERAGE explicitly or disable TEST_COVERAGE!\nYou need to install the python package coverage, e.g. with\n  pip install coverage\nor with\n  apt install python-coverage")
  endif()

  message(STATUS "Setting up test coverage")
  add_custom_target(coverage ${PYTHON_COVERAGE} combine --append .coverage plovasp/.coverage || true COMMAND ${PYTHON_COVERAGE} html COMMAND echo "Open ${CMAKE_BINARY_DIR}/test/htmlcov/index.html in browser!" WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/test)
endif()

enable_testing()

option(Build_Tests "Build the tests of the library " ON)
if (Build_Tests)
  message(STATUS "-------- Preparing tests  -------------")
 add_subdirectory(test)
endif()

#------------------------
# Documentation
#------------------------
option(Build_Documentation "Build documentation" OFF)
if(${Build_Documentation})
  if(NOT ${TRIQS_WITH_DOCUMENTATION})
    message("Error: TRIQS library has not been compiled with its documentation")
  endif()
  add_subdirectory(doc)
endif()

#--------------------------------------------------------
# Packaging
#--------------------------------------------------------
option(BUILD_DEBIAN_PACKAGE "Build a deb package" OFF)
if(BUILD_DEBIAN_PACKAGE)
  if(NOT CMAKE_INSTALL_PREFIX STREQUAL "/usr")
    message(FATAL_ERROR "CMAKE_INSTALL_PREFIX must be /usr for packaging")
  endif()
  SET(CPACK_GENERATOR "DEB")
  SET(CPACK_PACKAGE_VERSION ${DFT_TOOLS_VERSION})
  SET(CPACK_PACKAGE_CONTACT "https://github.com/TRIQS/dft_tools")
  EXECUTE_PROCESS(COMMAND dpkg --print-architecture OUTPUT_VARIABLE CMAKE_DEBIAN_PACKAGE_ARCHITECTURE OUTPUT_STRIP_TRAILING_WHITESPACE)
  SET(CPACK_DEBIAN_PACKAGE_DEPENDS "python, python-numpy, python-h5py, python-jinja2, python-mako, python-mpi4py, python-matplotlib, python-scipy, cpp2py (>= 1.5), triqs (>= 2.0)")
  SET(CPACK_DEBIAN_PACKAGE_SHLIBDEPS ON)
  SET(CPACK_DEBIAN_PACKAGE_GENERATE_SHLIBS ON)
  INCLUDE(CPack)
endif()
