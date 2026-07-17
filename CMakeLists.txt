# CMake build for the MOODS Python extension modules.
#
# This reproduces exactly what the old python/build.sh + python/setup.py did:
#   * run SWIG (C++, Python) on the core/*.i interface files
#   * compile the generated wrappers together with the core C++ sources into
#     the MOODS._tools / MOODS._scan / MOODS._parsers extension modules
#   * assemble the importable `MOODS` package (its __init__.py + the SWIG .py
#     wrappers + the compiled extensions)
#
# It is driven by pyproject.toml via scikit-build-core; there is no setup.py.

cmake_minimum_required(VERSION 3.15...3.30)
project(MOODS LANGUAGES CXX)

# --- C++ standard / PIC (was: --std=c++11 -fPIC in setup.py) ---------------
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# --- Toolchain -------------------------------------------------------------
find_package(SWIG REQUIRED)
include(UseSWIG)
find_package(Python REQUIRED COMPONENTS Interpreter Development.Module)

# --- Optimization flags (was: -O3 -march=native) ---------------------------
# Applied only when the compiler accepts them, so the build stays portable.
include(CheckCXXCompilerFlag)
set(MOODS_OPT_FLAGS "")
foreach(_flag -O3 -march=native)
  string(MAKE_C_IDENTIFIER "MOODS_HAVE${_flag}" _ok_var)
  check_cxx_compiler_flag("${_flag}" ${_ok_var})
  if(${_ok_var})
    list(APPEND MOODS_OPT_FLAGS "${_flag}")
  endif()
endforeach()

set(MOODS_CORE "${CMAKE_CURRENT_SOURCE_DIR}/core")

# C++ sources per module, mirroring the Extension() definitions in setup.py.
set(_src_common  "${MOODS_CORE}/moods_misc.cpp" "${MOODS_CORE}/match_types.cpp")
set(_src_tools   "${MOODS_CORE}/moods_tools.cpp")
set(_src_scan    "${MOODS_CORE}/moods_scan.cpp"
                 "${MOODS_CORE}/motif_0.cpp"
                 "${MOODS_CORE}/motif_h.cpp"
                 "${MOODS_CORE}/scanner.cpp"
                 "${MOODS_CORE}/moods_tools.cpp")
set(_src_parsers "${MOODS_CORE}/moods_parsers.cpp"
                 "${MOODS_CORE}/moods_tools.cpp")

foreach(mod tools scan parsers)
  set(_iface "${MOODS_CORE}/${mod}.i")
  set_property(SOURCE "${_iface}" PROPERTY CPLUSPLUS ON)
  set_property(SOURCE "${_iface}" PROPERTY INCLUDE_DIRECTORIES "${MOODS_CORE}")

  # Generates <mod>.py + the _<mod> extension (SWIG's Python naming), with the
  # .py wrapper written into <builddir>/MOODS/.
  swig_add_library(${mod}
    LANGUAGE python
    OUTPUT_DIR "${CMAKE_CURRENT_BINARY_DIR}/MOODS"
    SOURCES "${_iface}" ${_src_common} ${_src_${mod}})

  target_include_directories(${mod} PRIVATE "${MOODS_CORE}")
  target_compile_options(${mod} PRIVATE ${MOODS_OPT_FLAGS})
  target_link_libraries(${mod} PRIVATE Python::Module)

  # Install the compiled extension (_tools.so, …) and its .py wrapper into the
  # MOODS package inside the wheel.
  install(TARGETS ${mod} LIBRARY DESTINATION MOODS)
  install(FILES "${CMAKE_CURRENT_BINARY_DIR}/MOODS/${mod}.py" DESTINATION MOODS)
endforeach()

# Pure-Python package marker (kept verbatim from python/MOODS/__init__.py).
install(FILES "${CMAKE_CURRENT_SOURCE_DIR}/python/MOODS/__init__.py"
        DESTINATION MOODS)

# CLI script, mirroring setup.py's scripts=['scripts/moods-dna.py'].
install(PROGRAMS "${CMAKE_CURRENT_SOURCE_DIR}/python/scripts/moods-dna.py"
        DESTINATION "${SKBUILD_SCRIPTS_DIR}")
