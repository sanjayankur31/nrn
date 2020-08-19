# Add directory-level default compiler flags -- these should be added to all NEURON targets, but not
# targets from included projects like CoreNEURON and NMODL
add_compile_options(${NRN_COMPILE_FLAGS})
add_compile_definitions(${NRN_COMPILE_DEFS})
add_link_options(${NRN_LINK_FLAGS})

# =============================================================================
# List of python executables to build nrnpython against
# =============================================================================
if(NRN_ENABLE_PYTHON_DYNAMIC)
  if("${NRN_PYTHON_DYNAMIC}" STREQUAL "")
    set(NRN_PYTHON_EXE_LIST
        ${PYTHON_EXECUTABLE}
        CACHE INTERNAL "" FORCE)
  else()
    set(NRN_PYTHON_EXE_LIST
        ${NRN_PYTHON_DYNAMIC}
        CACHE INTERNAL "" FORCE)
  endif()
else()
  set(NRN_PYTHON_EXE_LIST
      ${PYTHON_EXECUTABLE}
      CACHE INTERNAL "" FORCE)
endif()

# =============================================================================
# rxdmath libraries (always build)
# =============================================================================
add_library(rxdmath SHARED ${CMAKE_CURRENT_SOURCE_DIR}/rxdmath.cpp)
if(NOT NRN_WINDOWS_BUILD)
    set_target_properties(rxdmath PROPERTIES SOVERSION 0.0.0)
endif()
install(TARGETS rxdmath DESTINATION ${LIB_INSTALL_DIR})

# =============================================================================
# nrnpython libraries (one lib per python)
# =============================================================================

set(nrnpython_lib_list)

# user has selected dynamic python support (could be multiple versions)
if(NRN_ENABLE_PYTHON_DYNAMIC)
  set(INCLUDE_DIRS
      .
      ../oc
      ../nrnoc
      ../ivoc
      ../nrniv
      ../ivos
      ../gnu
      ../mesch
      ../nrnmpi
      ${PROJECT_BINARY_DIR}/src/nrnpython
      ${PROJECT_BINARY_DIR}/src/ivos
      ${PROJECT_BINARY_DIR}/src/oc
      ${IV_INCLUDE_DIR})

  if(LINK_AGAINST_PYTHON)
    list(LENGTH NRN_PYTHON_EXE_LIST _num_pythons)
    math(EXPR num_pythons "${_num_pythons} - 1")
    foreach(val RANGE ${num_pythons})
      list(GET NRN_PYTHON_VER_LIST ${val} pyver)
      list(GET NRN_PYTHON_INCLUDE_LIST ${val} pyinc)
      list(GET NRN_PYTHON_LIB_LIST ${val} pylib)
      add_library(nrnpython${pyver} SHARED ${NRN_NRNPYTHON_SRC_FILES})
      target_include_directories(nrnpython${pyver} BEFORE PUBLIC ${pyinc} ${INCLUDE_DIRS})
      target_link_libraries(nrnpython${pyver} nrniv_lib ${pylib} ${Readline_LIBRARY})
      add_dependencies(nrnpython${pyver} nrniv_lib)
      list(APPEND nrnpython_lib_list nrnpython${pyver})
      install(TARGETS nrnpython${pyver} DESTINATION ${NRN_INSTALL_SHARE_LIB_DIR})
    endforeach()
  else()
    # build python3 library and install it
    if(NRNPYTHON_INCLUDE3)
      add_library(nrnpython3 SHARED ${NRN_NRNPYTHON_SRC_FILES})
      set_target_properties(nrnpython3 PROPERTIES SOVERSION 0.0.0)
      add_dependencies(nrnpython3 nrniv_lib)
      target_include_directories(nrnpython3 PRIVATE "${NRN_OC_GENERATED_SOURCES}")
      list(APPEND nrnpython_lib_list nrnpython3)
      target_include_directories(nrnpython3 BEFORE PUBLIC ${NRNPYTHON_INCLUDE3} ${INCLUDE_DIRS})
      install(TARGETS nrnpython3 DESTINATION ${LIB_INSTALL_DIR})
    endif()
  endif()
endif()

configure_file(_config_params.py.in "${PROJECT_BINARY_DIR}/lib/python/neuron/_config_params.py"
               @ONLY)

# Install package files that were created in build (e.g. .py.in)
install(
  DIRECTORY ${PROJECT_BINARY_DIR}/share/lib/python
  DESTINATION lib
  FILES_MATCHING
  PATTERN *.py
  PATTERN *.so
  PATTERN *.dylib
  PATTERN *.dat)

# =============================================================================
# If NEURON python module installation is enabled (CMake install)
# =============================================================================
if(NRN_ENABLE_MODULE_INSTALL)

  # Setup MinGW toolchain for setuptools
  if(MINGW)
    file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/setup.cfg "[build]\ncompiler=mingw32")

    # replace windows path of the form C:/msys64 to C:\msys64
    string(REPLACE ":/" ":\\\\" module_install_opts "${NRN_MODULE_INSTALL_OPTIONS}")
    set(NRN_MODULE_INSTALL_OPTIONS
        "${module_install_opts}"
        CACHE INTERNAL "" FORCE)
  endif()

  # Here and for the neuron/rxd/geometry3d extensions, we build the neuron module in lib/python
  set(NRN_PYTHON_BUILD_LIB
      ${PROJECT_BINARY_DIR}/lib/python
      CACHE INTERNAL "" FORCE)

  # ~~~
  # To tickle setup.py into actually rebuilding if dependencies change,
  # need to copy inithoc.cpp in from source to binary dir if any module
  # dependent changes and make a custom target as well.
  # ~~~

  set(binary_dir_filename ${CMAKE_CURRENT_BINARY_DIR}/inithoc.cpp)
  set(source_dir_filename ${CMAKE_CURRENT_SOURCE_DIR}/inithoc.cpp)
  set(inithoc_hdeps
      ${CMAKE_CURRENT_SOURCE_DIR}/../oc/nrnmpi.h ${CMAKE_CURRENT_BINARY_DIR}/../oc/nrnmpiuse.h
      ${CMAKE_CURRENT_BINARY_DIR}/../oc/nrnpthread.h ${CMAKE_CURRENT_BINARY_DIR}/nrnpython_config.h)

  add_custom_command(
    OUTPUT ${binary_dir_filename}
    COMMAND cp ${source_dir_filename} ${binary_dir_filename}
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    DEPENDS ${source_dir_filename} ${inithoc_hdeps})

  add_custom_target(
    hoc_module ALL
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    DEPENDS ${binary_dir_filename})

  # =============================================================================
  # Copy necessary files to build directory
  # =============================================================================

  add_custom_command(
    TARGET hoc_module
    PRE_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${PROJECT_SOURCE_DIR}/share/lib
            ${PROJECT_BINARY_DIR}/share/nrn/lib
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${PROJECT_SOURCE_DIR}/share/demo
            ${PROJECT_BINARY_DIR}/share/nrn/demo)

  add_dependencies(hoc_module nrniv_lib ${nrnpython_lib_list})
endif()
