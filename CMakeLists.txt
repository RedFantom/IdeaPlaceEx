cmake_minimum_required(VERSION 2.8)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON )
set(CMAKE_VERBOSE_MAKEFILE ON)

set(PROJECT_NAME "IdeaPlaceEx")
project(${PROJECT_NAME})

find_package (PkgConfig REQUIRED)
#The project version number
set(VERSION_MAJOR   0   CACHE STRING "Project major version number.")
set(VERSION_MINOR   2   CACHE STRING "Project minor version number.")
set(VERSION_PATCH   0   CACHE STRING "Project patch version number.")

# output path
set(EXECUTABLE_OUTPUT_PATH ${CMAKE_CURRENT_SOURCE_DIR}/bin)
set(DOCUMENT_OUTPUT_PATH ${CMAKE_CURRENT_SOURCE_DIR}/docs)
set(PROJECT_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/src)

#Check if intall prefix flag is found, if not, set a default path
if (CMAKE_INSTALL_PREFIX)
    set(CMAKE_INSTALL_PREFIX_DIR ${CMAKE_INSTALL_PREFIX})
else()
    set(CMAKE_INSTALL_PREFIX_DIR ${CMAKE_CURRENT_SOURCE_DIR}/install)
endif()
unset(CMAKE_INSTALL_PREFIX)


message(STATUS "Install prefix: ${CMAKE_INSTALL_PREFIX}")

# Find the directory for LIMBO. If cache variable LIMBO_DIR is defined, use it as the path to LIMBO, otherwise use the system LIMBO_DIR
SET(LIMBO_ROOT_DIR "../limbo/limbo")
include_directories(${LIMBO_ROOT_DIR}/..)
unset (LIMBO_DIR CACHE)

find_package(Boost 1.6 COMPONENTS system graph iostreams)
find_package( ZLIB REQUIRED )

# Find the directory for LPSOLVE. If cache variable LPSOLVE_DIR is defined, use it as the path to LPSOLVE, otherwise use the system LPSOLVE_DIR
if (LPSOLVE_DIR)
    set(LPSOLVE_ROOT_DIR ${LPSOLVE_DIR})
    message(STATUS "Use LPSOLVE: ${LPSOLVE_ROOT_DIR}")
    set(USE_LPSOLVE TRUE)
elseif(DEFINED ENV{LPSOLVE_DIR})
    set(LPSOLVE_ROOT_DIR $ENV{LPSOLVE_DIR})
    message(STATUS "Use system LPSOLVE ${LPSOLVE_ROOT_DIR}")
    set(USE_LPSOLVE TRUE)
else()
    message(STATUS "Disable lpsolve")
    set(USE_LPSOLVE FALSE)
endif()
unset (LPSOLVE_DIR CACHE)

if (USE_LPSOLVE)
find_library(LPSOLVE_LIBRARIES
    NAMES liblpsolve55.so
    PATHS ${LPSOLVE_ROOT_DIR}
    )
message(STATUS "LPSOLVE_ROOT_DIR = ${LPSOLVE_ROOT_DIR}")
message(STATUS "LPSOLVE_LIBRARIES = ${LPSOLVE_LIBRARIES}")
else()
    add_compile_definitions(LP_NOT_USE_LPSOLVE)
endif()

# add a target to generate API documentation with Doxygen
find_package(Doxygen)
option(BUILD_DOCUMENTATION "Create and install the HTML based API documentation (requires Doxygen)" ${DOXYGEN_FOUND})

if(BUILD_DOCUMENTATION)
    if(NOT DOXYGEN_FOUND)
        message(FATAL_ERROR "Doxygen is needed to build the documentation.")
    endif()

    set(doxy_main_page ${CMAKE_CURRENT_SOURCE_DIR}/Readme.md)
    set(doxyfile_in ${CMAKE_CURRENT_SOURCE_DIR}/Doxyfile)
    set(doxyfile ${DOCUMENT_OUTPUT_PATH}/Doxyfile)

    #request to configure the file
    configure_file(${doxyfile_in} ${doxyfile} @ONLY)

    add_custom_target(doc
        COMMAND ${DOXYGEN_EXECUTABLE} ${doxyfile}
        WORKING_DIRECTORY ${DOCUMENT_OUTPUT_PATH}
        COMMENT "Generating API documentation with Doxygen"
        VERBATIM)

    #install(DIRECTORY ${DOCUMENT_OUTPUT_PATH}/html DESTINATION ${CMAKE_INSTALL_PREFIX_DIR}/doc)
endif()
mark_as_advanced(VERSION_MAJOR VERSION_MINOR VERSION_PATCH)



file(GLOB SOURCES src/global/*.h    src/global/*.cpp
                  src/db/*.h        src/db/*.cpp
                  src/util/*.h      src/util/*.cpp 
                  src/parser/*.h    src/parser/*.cpp
                  src/writer/gdsii/*.h    src/writer/gdsii/*.cpp
                  src/place/*.h    src/place/*.cpp src/place/nlp/*.cpp
                  src/pinassign/*.h src/pinassign/*.cpp
                  src/main/IdeaPlaceEx.h src/main/IdeaPlaceEx.cpp)

file(GLOB EXE_SOURCES src/main/main.cpp)
file(GLOB PY_API_SOURCES src/api/*.cpp)

# linking libraries
file(GLOB STATIC_LIB
    ${LIMBO_ROOT_DIR}/parsers/gdsii/gdsdb/libgdsdb.a
    ${LIMBO_ROOT_DIR}/parsers/gdsii/stream/libgdsparser.a
    ${LIMBO_ROOT_DIR}/thirdparty/gzstream/libgzstream.a)

include_directories(".")

# Pybind11
find_package(pybind11 REQUIRED )
include_directories(${PYBIND11_INCLUDE_DIRS})
link_libraries(${PYBIND11_LIBS})
unset(PYBIND11_DIR CACHE)

# Python
pkg_check_modules(PYTHON REQUIRED python2=2.7)
include_directories(${PYTHON_INCLUDE_DIRS})
link_libraries(${PYTHON_LIBRARIES})
message(STATUS "Found Python2: " ${PYTHON_FOUND})

# Lemon
pkg_check_modules(LEMON REQUIRED lemon)
include_directories(${LEMON_INCLUDE_DIRS})
link_libraries(lemon)
message(STATUS "Found Lemon: " ${LEMON_FOUND})
message(STATUS "Lemon libraries: " ${LEMON_LIBRARIES})

# Eigen
pkg_check_modules(EIGEN REQUIRED eigen3)
include_directories(${EIGEN_INCLUDE_DIRS})
link_libraries(${EIGEN_LIBRARIES})
message(STATUS "Found eigen: " ${EIGEN_FOUND})

#Gurobi
if (GUROBI_HOME)
    set(GUROBI_ROOT_DIR ${GUROBI_HOME})
    message(STATUS "Use Gurobi ${GUROBI_ROOT_DIR}")
    set(USE_GUROBI TRUE)
elseif(DEFINED ENV{GUROBI_HOME})
    set(GUROBI_ROOT_DIR $ENV{GUROBI_HOME})
    message(STATUS "Use system Gurobi ${GUROBI_ROOT_DIR}")
    set(USE_GUROBI TRUE)
else()
    message(STATUS "Disable Gurobi")
    set(USE_GUROBI FALSE)
endif()
unset(GUROBI_HOME CACHE)


if (USE_GUROBI)
    find_library(GUROBI_CPP
        NAMES gurobi_c++
        HINTS ${GUROBI_ROOT_DIR}/lib
        )
    find_library(GUROBI_C
        NAMES gurobi90
        HINTS ${GUROBI_ROOT_DIR}/lib
        )
    set(GUROBI_LIBRARIES ${GUROBI_CPP} ${GUROBI_C})
    message(STATUS "GUROBI_ROOT_DIR = ${GUROBI_ROOT_DIR}")
    message(STATUS "GUROBI_LIBRARIES = ${GUROBI_LIBRARIES}")
else()
    add_compile_definitions(LP_NOT_USE_GUROBI)
endif()

#wnlib: conjugate gradient for reference
set (WNLIB_DIR ../wnlib )
if (WNLIB_DIR)
    set(WNLIB_ROOT_DIR ${WNLIB_DIR})
    message(STATUS "Use wnlib ${WNLIB_ROOT_DIR}")
    set(USE_WNLIB TRUE)
elseif(DEFINED ENV{WNLIB_DIR})
    set(WNLIB_ROOT_DIR $ENV{WNLIB_DIR})
    message(STATUS "Use system wnlib ${WNLIB_ROOT_DIR}")
    set(USE_WNLIB TRUE)
else()
    message(STATUS "Disable wnlib")
    set(USE_WNLIB FALSE)
endif()
unset(WNLIB_DIR CACHE)

find_library(WNLIB_LIBRARIES
    NAMES text.a
    HINTS ${WNLIB_ROOT_DIR}/acc
    )

#Print out the used build type
IF(CMAKE_BUILD_TYPE MATCHES Debug)
    message(STATUS "Using build type DEBUG")
    #ENDIF(CMAKE_BUILD_TYPE MATCHES Debug)
ELSEIF(CMAKE_BUILD_TYPE MATCHES Release)
    message(STATUS "Using build type RELEASE")
    #ENDIF(CMAKE_BUILD_TYPE MATCHES Release)
ELSEIF(CMAKE_BUILD_TYPE MATCHES Profile)
    message(STATUS "Using build type PROFILE")
    #ENDIF(CMAKE_BUILD_TYPE MATCHES Profile)
ELSE()
    message(STATUS "Using build type DEFAULT: using Release flag")
    set(CMAKE_BUILD_TYPE Release)
ENDIF()
set(CMAKE_CXX_FLAGS "-std=c++17 -Wall -fopenmp ")
set(CMAKE_CXX_FLAGS_DEBUG "-g -O0 -fno-inline ")
set(CMAKE_CXX_FLAGS_RELEASE "-O3")
set(CMAKE_CXX_FLAGS_PROFILE "-O3 -pg -Winline")
set(CMAKE_EXE_LINKER_FLAGS "-pthread -static-libgcc -static-libstdc++")


set(TO_LINK_LIBS ${STATIC_LIB} ${Boost_LIBRARIES} ${ZLIB_LIBRARIES})


set(TO_INCLUDE_PATH 
    ${CMAKE_CURRENT_SOURCE_DIR}/src
    ${Boost_INCLUDE_DIR}
    ${ZLIB_INCLUDE_DIRS}
    ${LIMBO_ROOT_DIR}/include
    ${PYTHON_INCLUDE_DIRS}
    ${PYBIND11_ROOT_DIR}/include
    ${LEMON_ROOT_DIR}/include
    ${EIGEN_ROOT_DIR}
)

if (USE_LPSOLVE)
    set(TO_LINK_LIBS ${TO_LINK_LIBS} ${LPSOLVE_LIBRARIES})
    set(TO_INCLUDE_PATH ${TO_INCLUDE_PATH} ${LPSOLVE_ROOT_DIR})
endif()

if (USE_GUROBI)
    set(TO_LINK_LIBS ${TO_LINK_LIBS} ${GUROBI_LIBRARIES})
    set(TO_INCLUDE_PATH ${TO_INCLUDE_PATH} ${GUROBI_ROOT_DIR}/include)
endif()

if (USE_WNLIB)
    link_libraries(${WNLIB_LIBRARIES})
    include_directories(${WNLIB_ROOT_DIR}/acc/h)
endif()

add_executable(${PROJECT_NAME} ${SOURCES} ${EXE_SOURCES})

include_directories(${TO_INCLUDE_PATH})
include_directories( "." )
pkg_check_modules( EIGEN REQUIRED eigen3 )
link_libraries(${EIGEN_LIBRARIES})
include_directories(${EIGEN_INCLUDE_DIRS})


target_link_libraries(${PROJECT_NAME} 
    ${TO_LINK_LIBS}
    )

## Add modules to pybind
pybind11_add_module("IdeaPlaceExPy" ${PY_API_SOURCES} ${SOURCES})
target_link_libraries("IdeaPlaceExPy" PUBLIC ${TO_LINK_LIBS}
)

##Install
#install (TARGETS ${PROJECT_NAME} DESTINATION ${CMAKE_INSTALL_PREFIX_DIR}/bin)
