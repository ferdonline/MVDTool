##############################################
# Requirements
find_package(PythonInterpLibs)
find_package(Cython)

option(BUILD_PYTHON_BINDINGS "Build python bindings?" ON)
if(BUILD_PYTHON_BINDINGS)

##############################################
# Definitions
set(cymod_src "${CMAKE_CURRENT_SOURCE_DIR}/mvdtool.pyx")
set(cybindings_cpp "${CMAKE_CURRENT_SOURCE_DIR}/mvdtool.cpp")
set(local_includes_dir "${CMAKE_CURRENT_SOURCE_DIR}/includes")
FILE(GLOB cy_srcs *.pyx *.pxi includes/*.pxd)

# Target properties, required for the extension
get_target_property(_link_libs mvd-tool LINK_LIBRARIES)
get_target_property(_include_dirs mvd-tool INCLUDE_DIRECTORIES)


##############################################
# Rules
set(OPT_REBUILD_OPTION_NAME REBUILD_PYTHON_BINDINGS)
set(OPT_REBUILD_SOURCE_FILE ${cybindings_cpp})
# Magic line to exec the checking
include(OptionRebuildSrcFile)
if(REBUILD_PYTHON_BINDINGS AND NOT CYTHON_FOUND)
    message(FATAL_ERROR "Cython is required to rebuild python bindings but was not found" )
endif()


# .pyx -> .cpp
add_custom_command(
    OUTPUT ${cybindings_cpp}
    COMMAND PYTHONPATH=${CMAKE_CURRENT_SOURCE_DIR} ${CYTHON_EXECUTABLE} ${CYTHON_FLAGS} --fast-fail --cplus -I${local_includes_dir} --output-file ${cybindings_cpp} ${cymod_src}
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    # No dependencies, except own sources
    DEPENDS ${cy_srcs}
    COMMENT "Cythonizing ${cymod_src}"
 )

# .cpp -> shared_lib
add_definitions("-std=c++11" "-Wfatal-errors" "-Wno-strict-aliasing")
python_add_module(mvdtool ${cybindings_cpp})
target_include_directories(mvdtool PUBLIC ${PYTHON_INCLUDE_DIRS} ${local_includes_dir})
target_link_libraries(mvdtool ${_link_libs})


##############################################
# Python packaging

set(_inc_str ${local_includes_dir})
foreach(dir ${_include_dirs})
    set(_inc_str "${_inc_str}:${dir}")
endforeach()

# Wheel
add_custom_target(pydist
    COMMAND python setup.py build --build-base=${CMAKE_CURRENT_BINARY_DIR}  build_ext "-I${_inc_str}" bdist_wheel --dist-dir ${CMAKE_BINARY_DIR}
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    DEPENDS ${cybindings_cpp}
    COMMENT "Building distributable Python wheel in ${CMAKE_BINARY_DIR}"
)

# Source dist - still requires include dirs
add_custom_target(pysrcdist
    COMMAND python setup.py build --build-base=${CMAKE_CURRENT_BINARY_DIR} sdist --dist-dir ${CMAKE_BINARY_DIR}
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    DEPENDS ${cybindings_cpp}
    COMMENT "Building distributable Python wheel in ${CMAKE_BINARY_DIR}"
)


##############################################
# Install
#
set(_default_prefixes "/usr/local" "/usr")
list(FIND _default_prefixes ${CMAKE_INSTALL_PREFIX} _index)
if(${_index} GREATER -1)
    # Default installations install Python to default site-packages
    set(_py_install_prefix ${PYTHON_SITE_PACKAGES})
else()
    set(_py_install_prefix ${CMAKE_INSTALL_PREFIX})
endif()

install(TARGETS mvdtool
        LIBRARY DESTINATION "${_py_install_prefix}" )


##############################################
# TESTS
#
add_test(NAME python_read_mvd2
         COMMAND bash -c "PYTHONPATH=${CMAKE_CURRENT_BINARY_DIR} python test/read_mvd2.py"
         WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
)

add_test(NAME python_read_mvd3
         COMMAND bash -c "PYTHONPATH=${CMAKE_CURRENT_BINARY_DIR} python test/read_mvd3.py"
         WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
)


endif() # /BUILD_PYTHON_BINDINGS
