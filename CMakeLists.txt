# CMake version check
cmake_minimum_required(VERSION 3.14 FATAL_ERROR)

#KS: Enable language, necessary when using CUDA
enable_language(CXX)
set(NucDeEx_VERSION 1.0.0)

project(NucDeEx VERSION ${NucDeEx_VERSION} LANGUAGES CXX)

# Changes default install path to be a subdirectory of the build dir.
# Can set build dir at configure time with -DCMAKE_INSTALL_PREFIX=/install/path
if(CMAKE_INSTALL_PREFIX STREQUAL "" OR CMAKE_INSTALL_PREFIX STREQUAL
  "/usr/local")
  set(CMAKE_INSTALL_PREFIX "${CMAKE_BINARY_DIR}")
elseif(NOT DEFINED CMAKE_INSTALL_PREFIX)
  set(CMAKE_INSTALL_PREFIX "${CMAKE_BINARY_DIR}")
endif()

# Use the compilers found in the path
find_program(CMAKE_C_COMPILER NAMES $ENV{CC} gcc PATHS ENV PATH NO_DEFAULT_PATH)
find_program(CMAKE_CXX_COMPILER NAMES $ENV{CXX} g++ PATHS ENV PATH NO_DEFAULT_PATH)

LIST(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake/Modules)
################################## Dependencies ################################

include(CPM)

#Luke's handing cmake modules which Neutrino hep experiments might want
CPMFindPackage(
      NAME CMakeModules
      GIT_TAG stable
      GITHUB_REPOSITORY NuHepMC/CMakeModules
      DOWNLOAD_ONLY
  )
include(${CMakeModules_SOURCE_DIR}/NuHepMCModules.cmake)
include(NuHepMCUtils)

include(ROOT)
if(NOT TARGET ROOT::ROOT)
  cmessage(FATAL_ERROR "NucDeEx Expected dependency target: ROOT::ROOT")
endif()

find_package(NEUT QUIET)

set(NucDeEx_NEUT_ENABLED TRUE)
if(NOT TARGET NEUT::NEUT)
  cmessage(STATUS "Didn't find NEUT")
  set(NucDeEx_NEUT_ENABLED FALSE)
endif()

############################  C++ Compiler  ####################################
if (NOT DEFINED CMAKE_CXX_STANDARD OR "${CMAKE_CXX_STANDARD} " STREQUAL " ")
  set(CMAKE_CXX_STANDARD 11)
endif()

# KS: If C++ standard is lower than C++ standard used for ROOT compilation things will go terribly wrong
if(DEFINED ROOT_CXX_STANDARD AND ROOT_CXX_STANDARD GREATER CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD ${ROOT_CXX_STANDARD})
endif()
cmessage(STATUS "CMAKE CXX Standard: ${CMAKE_CXX_STANDARD}")

# KS: ROOT changed cmake in 6.32, we should move avay from using Luke's hack, keep it for due to compatibility
if(ROOT_CXX_STANDARD LESS 14 AND ROOT_VERSION VERSION_LESS 6.32.00)
  cmessage(WARNING "ROOT CXX STANDARD: ${ROOT_CXX_STANDARD}")
endif()

############################  Setting Flags  ####################################
# Check the where to install 
cmessage(STATUS "CMAKE_INSTALL_PREFIX: \"${CMAKE_INSTALL_PREFIX}\"")

# add cmake script files
if(NOT ${CMAKE_SCRIPT_SETUP})  #Check if setup by Experiment MaCh3
  set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")
endif()

# Custom commands
set( CMAKE_EXPORT_COMPILE_COMMANDS ON )

add_compile_options(
    -Wno-deprecated
    -g -Wall
    -ggdb3
    -fPIC
    -O2
)


################################# Build NucDeEx ##################################

set(HEADERS_NucDeEx
    include/NucDeExConsts.hh
    include/NucDeExDeexcitation.hh
    include/NucDeExDeexcitationBase.hh
    include/NucDeExDeexcitationPhole.hh
    include/NucDeExDeexcitationTALYS.hh
    include/NucDeExEventInfo.hh
    include/NucDeExNucleus.hh
    include/NucDeExNucleusTable.hh
    include/NucDeExParticle.hh
    include/NucDeExRandom.hh
    include/NucDeExUtils.hh
    include/ReadTALYS.hh
    )

add_library(NucDeEx SHARED
    $<$<BOOL:${NucDeEx_NEUT_ENABLED}>:NEUTNucDeExInterace.cc>
    src/NucDeExNucleus.cc
    src/NucDeExRandom.cc
    src/NucDeExDeexcitationBase.cc
    src/NucDeExDeexcitation.cc
    src/NucDeExDeexcitationPhole.cc
    src/NucDeExDeexcitationTALYS.cc
    src/NucDeExEventInfo.cc
    src/NucDeExNucleusTable.cc
    src/NucDeExParticle.cc
    src/NucDeExUtils.cc
    src/ReadTALYS.cc
    )

set_target_properties(NucDeEx PROPERTIES
    PUBLIC_HEADER "${HEADERS_NucDeEx}"
    EXPORT_NAME NucDeEx)

target_link_libraries(NucDeEx ROOT::ROOT)
target_include_directories(NucDeEx PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/>
  $<INSTALL_INTERFACE:include>)


install(TARGETS NucDeEx
        EXPORT NucDeEx-targets
        LIBRARY DESTINATION lib/
        PUBLIC_HEADER DESTINATION include/
        )

add_library(NucDeEx::All ALIAS NucDeEx)

add_subdirectory(main)

############################  Install  ####################################
include(CMakePackageConfigHelpers)

write_basic_package_version_file(${CMAKE_BINARY_DIR}/NucDeExConfigVersion.cmake
VERSION ${NucDeEx_VERSION}
COMPATIBILITY AnyNewerVersion)
