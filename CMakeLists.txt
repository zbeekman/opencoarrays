cmake_minimum_required(VERSION 3.0)

#Name project and specify source languages
project(opencoarrays VERSION 1.0 LANGUAGES C Fortran)

#Print an error message on an attempt to build inside the source directory tree:
if ("${CMAKE_CURRENT_SOURCE_DIR}" STREQUAL "${CMAKE_CURRENT_BINARY_DIR}")
  message(FATAL_ERROR "ERROR! "
    "CMAKE_CURRENT_SOURCE_DIR=${CMAKE_CURRENT_SOURCE_DIR}"
    " == CMAKE_CURRENT_BINARY_DIR=${CMAKE_CURRENT_BINARY_DIR}"
    "\nThis archive does not support in-source builds:\n"
    "You must now delete the CMakeCache.txt file and the CMakeFiles/ directory under"
    "the 'src' source directory or you will not be able to configure correctly!"
    "\nYou must now run something like:\n"
    "  $ rm -r CMakeCache.txt CMakeFiles/"
    "\n"
    "Please create a directory outside the opencoarrays source tree and build under that outside directory "
    "in a manner such as\n"
    "  $ mkdir build-opencarrays\n"
    "  $ cd build-opencoarrays\n"
    "  $ CC=mpicc FC=mpif90 cmake <path-to-opencoarrays-source-directory> -DCMAKE_INSTALL_PREFIX=<path-to-install-directory>\n"
    "\nsubstituting the appropriate syntax for your shell (the above line assumes the bash shell)."
    )
endif()

#Report untested Fortran compiler unless explicitly directed to build all examples.
if ("${CMAKE_Fortran_COMPILER_ID}" MATCHES "GNU" )
  add_definitions(-DPREFIX_NAME=_gfortran_caf_)
else()
  message(WARNING 
    "*** Attempting to build with untested Fortran compiler: ${CMAKE_Fortran_COMPILER_ID}."
    "*** Please report any failures to opencoarrays@googlegroups.com"
  )
endif()

include(CheckFortranSourceCompiles.cmake)
CHECK_Fortran_SOURCE_COMPILES("
  program main
    implicit none
    integer :: i
    i = this_image()
  end program
" Check_Simple_Coarray_Fortran_Source_Compiles)

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/src)

add_subdirectory(src)

install(EXPORT OpenCoarraysTargets 
  NAMESPACE
    OpenCoarrays::
  DESTINATION
    lib/cmake/opencoarrays
)
include(CMakePackageConfigHelpers)
write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/OpenCoarraysConfigVersion.cmake"
  VERSION 1.0
  COMPATIBILITY AnyNewerVersion
)
configure_file(OpenCoarraysConfig.cmake.in CMakeFiles/OpenCoarraysConfig.cmake @ONLY)

install(
  FILES
    ${CMAKE_CURRENT_BINARY_DIR}/CMakeFiles/OpenCoarraysConfig.cmake
    "${CMAKE_CURRENT_BINARY_DIR}/OpenCoarraysConfigVersion.cmake"
  DESTINATION
    lib/cmake/opencoarrays
)

add_library(OpenCoarrays INTERFACE)
target_compile_options(OpenCoarrays INTERFACE -fcoarray=lib)
target_link_libraries(OpenCoarrays INTERFACE caf_mpi)

install(DIRECTORY  ${CMAKE_BINARY_DIR}/mod DESTINATION .)

enable_testing()

function(add_mpi_test name num_mpi_proc path)
   set(test_parameters ${MPIEXEC_NUMPROC_FLAG} ${num_mpi_proc} )
   add_test(NAME ${name} COMMAND ${MPIEXEC} ${test_parameters} "${path}")
   set_property(TEST ${name} PROPERTY PASS_REGULAR_EXPRESSION "Test passed.")
endfunction(add_mpi_test)

set(tests_root ${CMAKE_CURRENT_BINARY_DIR}/src/tests)

# Unit tests targeting each libcaf_mpi function, argument, and branch of code
add_mpi_test(initialize_mpi 2 ${tests_root}/unit/init_register/initialize_mpi)
add_mpi_test(register 2 ${tests_root}/unit/init_register/register)
add_mpi_test(register_rename_me 2 ${tests_root}/unit/init_register/register_rename_me)
add_mpi_test(register_rename_me_too 2 ${tests_root}/unit/init_register/register_rename_me_too)
add_mpi_test(allocate_as_barrier 2 ${tests_root}/unit/init_register/allocate_as_barrier)
add_mpi_test(allocate_as_barrier_proc 2 ${tests_root}/unit/init_register/allocate_as_barrier_proc)
add_mpi_test(get_array 2 ${tests_root}/unit/send-get/get_array)
add_mpi_test(send_array 2 ${tests_root}/unit/send-get/send_array)
add_mpi_test(get_with_offset_1d 2 ${tests_root}/unit/send-get/get_with_offset_1d)
add_mpi_test(whole_get_array 2 ${tests_root}/unit/send-get/whole_get_array)
add_mpi_test(strided_get 2 ${tests_root}/unit/send-get/strided_get)

# Integration tests verifying the use of libcaf_mpi in applications
add_mpi_test(hello_multiverse 2 ${tests_root}/integration/coarrayHelloWorld/hello_multiverse)
add_mpi_test(coarray_burgers_pde 2 ${tests_root}/integration/pde_solvers/coarrayBurgers/coarray_burgers_pde)
add_mpi_test(co_heat 2 ${tests_root}/integration/pde_solvers/coarrayHeatSimplified/co_heat)
add_mpi_test(tally_image_numbers 4 ${tests_root}/integration/extensions/tally_image_numbers)

if ("${CMAKE_SYSTEM_NAME}" MATCHES "Linux" )
  add_mpi_test(coarray_navier_stokes 2 ${tests_root}/integration/pde_solvers/navier-stokes/coarray_navier_stokes)
  set_property(TEST coarray_navier_stokes PROPERTY PASS_REGULAR_EXPRESSION "Test passed.")
endif()