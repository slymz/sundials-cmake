#
# author: slymz 
# See: https://github.com/slymz/sundials-cmake
# Updated (and simplified) alternative to Sundials distro CMakeLists.txt
# See below for original licence/version text
#

# ---------------------------------------------------------------
# $Revision: 1.10 $
# $Date: 2010/12/15 22:28:16 $
# ---------------------------------------------------------------
# Programmer:  Radu Serban @ LLNL
# ---------------------------------------------------------------
# Copyright (c) 2007, The Regents of the University of California.
# Produced at the Lawrence Livermore National Laboratory.
# All rights reserved.
# For details, see the LICENSE file.
# ---------------------------------------------------------------
# Top level CMakeLists.txt for SUNDIALS (for cmake build system)


# -------------------------------------------------------------
# Initial commands
# -------------------------------------------------------------
PROJECT(sundials C)
CMAKE_MINIMUM_REQUIRED(VERSION 2.8.4)

# Set some variables with info on the SUNDIALS project

# Only PACKAGE_VERSION is being used
#SET(PACKAGE_BUGREPORT "radu@llnl.gov")
#SET(PACKAGE_NAME "SUNDIALS")
#SET(PACKAGE_STRING "SUNDIALS 2.4.0")
#SET(PACKAGE_TARNAME "sundials")
SET(PACKAGE_VERSION "2.4.0")

# Prohibit in-source build
IF("${CMAKE_SOURCE_DIR}" STREQUAL "${CMAKE_BINARY_DIR}")
  MESSAGE(FATAL_ERROR "In-source build prohibited.")
ENDIF("${CMAKE_SOURCE_DIR}" STREQUAL "${CMAKE_BINARY_DIR}")

# Specify the VERSION and SOVERSION for shared libraries
SET(cvodelib_VERSION "1.0.0")
SET(cvodelib_SOVERSION "1")

SET(cvodeslib_VERSION "2.0.0")
SET(cvodeslib_SOVERSION "2")

SET(idalib_VERSION "2.0.0")
SET(idalib_SOVERSION "2")

SET(idaslib_VERSION "0.0.0")
SET(idaslib_SOVERSION "0")

SET(kinsollib_VERSION "1.0.0")
SET(kinsollib_SOVERSION "1")

SET(cpodeslib_VERSION "0.0.0")
SET(cpodeslib_SOVERSION "0")

SET(nveclib_VERSION "0.0.2")
SET(nveclib_SOVERSION "0")


# -------------------------------------------------------------
# Which modules to build?
# -------------------------------------------------------------

# Pick those that a directory exists right under src
# For those that doesn't force turn off 

foreach ( module cvode cvodes ida idas kinsol cpodes )
   string(TOUPPER ${module} MODULE)
   IF(IS_DIRECTORY "${PROJECT_SOURCE_DIR}/src/${module}")
      OPTION(BUILD_${MODULE}  "Build the ${MODULE} library"  ON)
   ELSE()
      set( BUILD_${MODULE} OFF CACHE BOOL 
         "Source for optional ${MODULE} library not available. \
          Download at https://computation.llnl.gov/casc/sundials/download/download.html to enable."  
         FORCE )
   ENDIF()
endforeach()

# -------------------------------------------------------------
# Other configuration options
# -------------------------------------------------------------

OPTION(BUILD_SHARED_LIBS "Build shared libraries? Turn off to build static libs." ${TARGET_SUPPORTS_SHARED_LIBS})
SET(SUNDIALS_PRECISION "double" CACHE STRING "double, single or extended")


# prepare substitution variable PRECISION_LEVEL for sundials_config.h
STRING(TOUPPER ${SUNDIALS_PRECISION} SUNDIALS_PRECISION)
SET(PRECISION_LEVEL "#define SUNDIALS_${SUNDIALS_PRECISION}_PRECISION 1")

# Option to use the generic math libraries (UNIX only)
# ----------------------------------------------------

#SET(GENERIC_MATH_LIB "#define SUNDIALS_USE_GENERIC_MATH")
#IF(UNIX)
#  OPTION(USE_GENERIC_MATH "Use generic (std-c) math libraries" ON)
#  IF(USE_GENERIC_MATH)
#    # executables will be linked against -lm 
#    SET(EXTRA_LINK_LIBS -lm)
#    # prepare substitution variable GENERIC_MATH_LIB for sundials_config.h
#    SET(GENERIC_MATH_LIB "#define SUNDIALS_USE_GENERIC_MATH")
#  ENDIF(USE_GENERIC_MATH)
#ENDIF(UNIX)

# -------------------------------------------------------------
# Enable Fortran support?
# -------------------------------------------------------------

# FCMIX support is an option only if at least one solver that 
# provides such an interface is built.

IF(BUILD_CVODE OR BUILD_IDA OR BUILD_KINSOL)
  set( FCMIX_ENABLE "OFF" CACHE BOOL "Enable Fortran-C support") 
ENDIF()

# -------------------------------------------------------------
# Enable BLAS/LAPACK support?
# -------------------------------------------------------------

OPTION(LAPACK_ENABLE "Enable Lapack support" OFF)

# -------------------------------------------------------------
# Enable MPI support?
# -------------------------------------------------------------

OPTION(MPI_ENABLE "Enable MPI support" OFF)

# -------------------------------------------------------------
# Enable examples?
# -------------------------------------------------------------

OPTION(EXAMPLES_ENABLE "Build the SUNDIALS examples" OFF)

IF(EXAMPLES_ENABLE)

  # Check if example files are to be exported
  OPTION( EXAMPLES_INSTALL "Install example files" ON )

   # If examples are to be exported, check where we should install them.
   IF(EXAMPLES_INSTALL)
      SET(EXAMPLES_INSTALL_PATH "${CMAKE_INSTALL_PREFIX}/examples" 
         CACHE PATH "Output directory for installing example files" )
   ENDIF()

ENDIF()

# -------------------------------------------------------------
# Add any other necessary compiler flags & definitions
# -------------------------------------------------------------

# Under Windows, add compiler directive to inhibit warnings
# about use of unsecure functions

IF(WIN32)
  ADD_DEFINITIONS(-D_CRT_SECURE_NO_WARNINGS)
ENDIF(WIN32)

# -------------------------------------------------------------
# A Fortran compiler is needed if:
# (a) FCMIX is enabled
# (b) LAPACK is enabled (for the name-mangling scheme)
# -------------------------------------------------------------

IF(FCMIX_ENABLE OR LAPACK_ENABLE)
   enable_language( Fortran )
   include( FortranCInterface )
   set( F77_MANGLE_MACRO1 "#define SUNDIALS_F77_FUNC${FortranCInterface_GLOBAL_MACRO}" )
   set( F77_MANGLE_MACRO2 "#define SUNDIALS_F77_FUNC_${FortranCInterface_GLOBAL__MACRO}" )
ENDIF()


# -------------------------------------------------------------
# Find (and test) the Lapack libraries
# -------------------------------------------------------------

# If LAPACK is needed, first try to find the appropriate
# libraries and linker flags needed to link against them.

# Macro to be inserted in sundials_config.h
SET(BLAS_LAPACK_MACRO "#define SUNDIALS_BLAS_LAPACK 0")

IF(LAPACK_ENABLE)

   find_package( BLAS   REQUIRED )
   find_package( LAPACK REQUIRED )
   
   IF(LAPACK_FOUND)
     SET(BLAS_LAPACK_MACRO "#define SUNDIALS_BLAS_LAPACK 1")
   ENDIF(LAPACK_FOUND)

ENDIF(LAPACK_ENABLE)

# -------------------------------------------------------------
# Decide how to compile MPI codes.
# -------------------------------------------------------------

IF(MPI_ENABLE)

   find_package( MPI REQUIRED )
   
   if( MPI_C_FOUND )
      set( CMAKE_REQUIRED_LIBRARIES ${MPI_C_LIBRARIES})
      include(CheckFunctionExists)
      CHECK_FUNCTION_EXISTS( mpi_comm_f2c X )
      if (X)
         SET(F77_MPI_COMM_F2C "#define SUNDIALS_MPI_COMM_F2C 1")
      else
         SET(F77_MPI_COMM_F2C "#define SUNDIALS_MPI_COMM_F2C 0")
      endif()
   endif()
ENDIF()

# -------------------------------------------------------------
# Configure the header file sundials_config.h
# -------------------------------------------------------------

# All required substitution variables should be available at this point.
# Generate the header file and place it in the binary dir.
CONFIGURE_FILE(
  ${PROJECT_SOURCE_DIR}/include/sundials/sundials_config.in
  ${PROJECT_BINARY_DIR}/include/sundials/sundials_config.h
  @ONLY
  )

# Add the include directory in the source tree and the one in
# the binary tree (for the header file sundials_config.h)
INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR}/include ${PROJECT_BINARY_DIR}/include )

# -------------------------------------------------------------
# Add selected modules to the build system
# -------------------------------------------------------------

# Shared components

ADD_SUBDIRECTORY(src/sundials)
ADD_SUBDIRECTORY(src/nvec_ser)
IF(MPI_C_FOUND)
  ADD_SUBDIRECTORY(src/nvec_par)
ENDIF(MPI_C_FOUND)

# CVODE library

IF(BUILD_CVODE)	
  ADD_SUBDIRECTORY(src/cvode)
  IF(FCMIX_ENABLE AND F77_FOUND)
    ADD_SUBDIRECTORY(src/cvode/fcmix)
  ENDIF()
ENDIF(BUILD_CVODE)

# CVODES library

IF(BUILD_CVODES)	
  ADD_SUBDIRECTORY(src/cvodes)
ENDIF(BUILD_CVODES)

# IDA library

IF(BUILD_IDA)	
  ADD_SUBDIRECTORY(src/ida)
  IF(FCMIX_ENABLE AND F77_FOUND)
    ADD_SUBDIRECTORY(src/ida/fcmix)
  ENDIF()
ENDIF(BUILD_IDA)

# IDAS library

IF(BUILD_IDAS)	
  ADD_SUBDIRECTORY(src/idas)
ENDIF(BUILD_IDAS)

# KINSOL library

IF(BUILD_KINSOL)	
  ADD_SUBDIRECTORY(src/kinsol)
  IF(FCMIX_ENABLE AND F77_FOUND)
    ADD_SUBDIRECTORY(src/kinsol/fcmix)
  ENDIF()
ENDIF(BUILD_KINSOL)

# CPODES library

IF(BUILD_CPODES)	
  ADD_SUBDIRECTORY(src/cpodes)
ENDIF(BUILD_CPODES)

# -------------------------------------------------------------
# Include the subdirectories corresponding to various examples
# -------------------------------------------------------------

IF(EXAMPLES_ENABLE)

  IF(EXAMPLES_INSTALL)
    SET(SHELL "sh")
    SET(prefix "${CMAKE_INSTALL_PREFIX}")
    SET(exec_prefix "${CMAKE_INSTALL_PREFIX}")
    SET(includedir "${prefix}/include")
    SET(libdir "${exec_prefix}/lib")
    SET(CPP "${CMAKE_C_COMPILER}")
    SET(CC "${CMAKE_C_COMPILER}")
    SET(CPPFLAGS "${CMAKE_C_FLAGS_RELEASE}")
    SET(CFLAGS "${CMAKE_C_FLAGS_RELEASE}")
    SET(LDFLAGS "${CMAKE_EXE_LINKER_FLAGS_RELEASE}")
    LIST2STRING(EXTRA_LINK_LIBS LIBS)
    IF(F77_FOUND)
      SET(F77 "${CMAKE_Fortran_COMPILER}")
      SET(F77_LNKR "${CMAKE_Fortran_COMPILER}")
      SET(FFLAGS "${CMAKE_Fortran_FLAGS_RELEASE}")
      SET(F77_LDFLAGS "${CMAKE_Fortran_FLAGS_RELEASE}")
      LIST2STRING(EXTRA_LINK_LIBS F77_LIBS)
    ENDIF(F77_FOUND)
    IF(LAPACK_FOUND)
      LIST2STRING(LAPACK_LIBRARIES BLAS_LAPACK_LIBS)
      SET(BLAS_LAPACK_LIBS "${LAPACK_LINKER_FLAGS} ${BLAS_LAPACK_LIBS}")
    ENDIF(LAPACK_FOUND)
    IF(MPIC_FOUND)
      IF(MPI_MPICC)
        SET(MPICC "${MPI_MPICC}")
        SET(MPI_INC_DIR ".")
        SET(MPI_LIB_DIR ".")
        SET(MPI_LIBS "")
        SET(MPI_FLAGS "")
      ELSE(MPI_MPICC)
        SET(MPICC "${CMAKE_C_COMPILER}")
        SET(MPI_INC_DIR "${MPI_INCLUDE_PATH}")
        SET(MPI_LIB_DIR ".")
        LIST2STRING(MPI_LIBRARIES MPI_LIBS)
      ENDIF(MPI_MPICC)
    ENDIF(MPIC_FOUND)
    IF(MPIF_FOUND)
      IF(MPI_MPIF77)
        SET(MPIF77 "${MPI_MPIF77}")
        SET(MPIF77_LNKR "${MPI_MPIF77}")
      ELSE(MPI_MPIF77)
        SET(MPIF77 "${CMAKE_Fortran_COMPILER}")
        SET(MPIF77_LNKR "${CMAKE_Fortran_COMPILER}")
        SET(MPI_INC_DIR "${MPI_INCLUDE_PATH}")
        SET(MPI_LIB_DIR ".")
        LIST2STRING(MPI_LIBRARIES MPI_LIBS)
      ENDIF(MPI_MPIF77)
    ENDIF(MPIF_FOUND)
  ENDIF(EXAMPLES_INSTALL)

  IF(BUILD_CVODE)
    ADD_SUBDIRECTORY(examples/cvode/serial)
    IF(FCMIX_ENABLE AND F77_FOUND)
      ADD_SUBDIRECTORY(examples/cvode/fcmix_serial)
    ENDIF(FCMIX_ENABLE AND F77_FOUND)
    IF(MPIC_FOUND)
      ADD_SUBDIRECTORY(examples/cvode/parallel)
    ENDIF(MPIC_FOUND)
    IF(MPIF_FOUND)
      ADD_SUBDIRECTORY(examples/cvode/fcmix_parallel)
    ENDIF(MPIF_FOUND)
  ENDIF(BUILD_CVODE)
  
  IF(BUILD_CVODES)	
    ADD_SUBDIRECTORY(examples/cvodes/serial)
    IF(MPIC_FOUND)
      ADD_SUBDIRECTORY(examples/cvodes/parallel)
    ENDIF(MPIC_FOUND)
  ENDIF(BUILD_CVODES)
  
  IF(BUILD_IDA)
    ADD_SUBDIRECTORY(examples/ida/serial)
    IF(FCMIX_ENABLE AND F77_FOUND)
      ADD_SUBDIRECTORY(examples/ida/fcmix_serial)
    ENDIF(FCMIX_ENABLE AND F77_FOUND)
    IF(MPIC_FOUND)
      ADD_SUBDIRECTORY(examples/ida/parallel)
    ENDIF(MPIC_FOUND)
    IF(MPIF_FOUND)
      ADD_SUBDIRECTORY(examples/ida/fcmix_parallel)
    ENDIF(MPIF_FOUND)
  ENDIF(BUILD_IDA)
  
  IF(BUILD_IDAS)	
    ADD_SUBDIRECTORY(examples/idas/serial)
    IF(MPIC_FOUND)
      ADD_SUBDIRECTORY(examples/idas/parallel)
    ENDIF(MPIC_FOUND)
  ENDIF(BUILD_IDAS)

  IF(BUILD_KINSOL)	
    ADD_SUBDIRECTORY(examples/kinsol/serial)
    IF(FCMIX_ENABLE AND F77_FOUND)
      ADD_SUBDIRECTORY(examples/kinsol/fcmix_serial)
    ENDIF(FCMIX_ENABLE AND F77_FOUND)
    IF(MPIC_FOUND)
      ADD_SUBDIRECTORY(examples/kinsol/parallel)
    ENDIF(MPIC_FOUND)
    IF(MPIF_FOUND)
      ADD_SUBDIRECTORY(examples/kinsol/fcmix_parallel)
    ENDIF(MPIF_FOUND)
  ENDIF(BUILD_KINSOL)

  IF(BUILD_CPODES)	
    ADD_SUBDIRECTORY(examples/cpodes/serial)
    IF(MPIC_FOUND)
      ADD_SUBDIRECTORY(examples/cpodes/parallel)
    ENDIF(MPIC_FOUND)
  ENDIF(BUILD_CPODES)
  
ENDIF(EXAMPLES_ENABLE)

#----------------------------------
# Install configuration header file
#----------------------------------

# install configured header file
INSTALL(
  FILES ${PROJECT_BINARY_DIR}/include/sundials/sundials_config.h
  DESTINATION include/sundials
  )
