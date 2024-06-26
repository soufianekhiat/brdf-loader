project(brdf-loader)
cmake_minimum_required(VERSION 3.18...3.22)

if (NOT SKBUILD)
  message(WARNING "\
  This CMake file is meant to be executed using 'scikit-build'. Running
  it directly will almost certainly not produce the desired result. If
  you are a user trying to install this package, please use the command
  below, which will install all necessary build dependencies, compile
  the package in an isolated environment, and then install it.
  =====================================================================
   $ pip install .
  =====================================================================
  If you are a software developer, and this is your own package, then
  it is usually much more efficient to install the build dependencies
  in your environment once and use the following command that avoids
  a costly creation of a new virtual environment at every compilation:
  =====================================================================
   $ python setup.py install
  =====================================================================")
endif()

# The following boilerplate code configures CMake so that it finds the right
# Python version (in particular, when it is run as part of 'scikit-build' using
# the GitHub Actions continuous integration server)
if (SKBUILD)
  # Fix missing shared library name for cibuildwheel+windows+pypy3.9
  if (MSVC AND NOT PYTHON_LIBRARY AND (${PYTHON_VERSION_STRING} MATCHES "3.9."))
    get_filename_component(PYTHON_LIBRARY ${PYTHON_INCLUDE_DIR} DIRECTORY)
    set(PYTHON_LIBRARY "${PYTHON_LIBRARY}/libs/python39.lib")
  endif()

  set(Python_VERSION "${PYTHON_VERSION_STRING}")
  set(Python_EXECUTABLE "${PYTHON_EXECUTABLE}")
  set(Python_INCLUDE_DIR "${PYTHON_INCLUDE_DIR}")
  set(Python_LIBRARY "${PYTHON_LIBRARY}")
elseif (MSVC)
  # MSVC needs a little extra help finding the Python library
  find_package(PythonInterp)
  find_package(Python)
endif()

# Create CMake targets for all Python components needed by nanobind
if (CMAKE_VERSION VERSION_GREATER_EQUAL 3.26)
  find_package(Python 3.8 COMPONENTS Interpreter Development.Module Development.SABIModule REQUIRED)
else()
  find_package(Python 3.8 COMPONENTS Interpreter Development.Module REQUIRED)
endif()

# Run `nanobind.cmake_dir()` from Python to detect where nanobind is installed
execute_process(
  COMMAND "${Python_EXECUTABLE}" -c "import nanobind; print(nanobind.cmake_dir())"
  OUTPUT_STRIP_TRAILING_WHITESPACE OUTPUT_VARIABLE NB_DIR)
list(APPEND CMAKE_PREFIX_PATH "${NB_DIR}")

# Now, import nanobind through CMake's find_package mechanism
find_package(nanobind CONFIG REQUIRED)

# We are now ready to compile the actual extension module
nanobind_add_module(
  # Name of the extension
  brdf_loader_ext

  # Target the stable ABI for Python 3.12+, which reduces
  # the number of binary wheels that must be built. This
  # does nothing on older Python versions
  STABLE_ABI

  # Build libnanobind statically and merge it into the
  # extension (which itself remains a shared library)
  #
  # If your project builds multiple extensions, you can
  # replace this flag by NB_SHARED to conserve space by
  # reusing a shared libnanobind across libraries
  NB_STATIC

  # Source code goes here
  python/brdf_loader_ext.cpp
)

target_include_directories(brdf_loader_ext
    PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}
)

# Install directive for scikit-build
install(TARGETS brdf_loader_ext LIBRARY DESTINATION .)
