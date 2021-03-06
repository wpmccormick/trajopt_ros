cmake_minimum_required(VERSION 2.8.3)
project(trajopt_sco)

add_compile_options(-std=c++11 -Wall -Wextra)

find_package(catkin REQUIRED COMPONENTS trajopt_utils)

find_package(Eigen3 REQUIRED)

set(SCO_SOURCE_FILES
    src/solver_interface.cpp
    src/modeling.cpp
    src/expr_ops.cpp
    src/expr_vec_ops.cpp
    src/optimizers.cpp
    src/modeling_utils.cpp
    src/num_diff.cpp
)

if (NOT APPLE)
  set (HAVE_BPMPD TRUE)
endif()

catkin_package(
  INCLUDE_DIRS include
  LIBRARIES ${PROJECT_NAME}
  CATKIN_DEPENDS trajopt_utils
#  DEPENDS system_lib
)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/Modules/")
find_package(GUROBI QUIET)

include_directories(
  include
  ${catkin_INCLUDE_DIRS}
  SYSTEM ${EIGEN3_INCLUDE_DIRS}
)

if (HAVE_BPMPD)
  add_executable(bpmpd_caller src/bpmpd_caller.cpp)

  if( CMAKE_SIZEOF_VOID_P EQUAL 8 ) # 64 bits
        set(BPMPD_LIBRARY "${CMAKE_CURRENT_SOURCE_DIR}/3rdpartylib/bpmpd_linux64.a")
  else()
        set(BPMPD_LIBRARY "${CMAKE_CURRENT_SOURCE_DIR}/3rdpartylib/bpmpd_linux32.a")
  endif()

  target_link_libraries(bpmpd_caller ${BPMPD_LIBRARY} -static)

  list(APPEND SCO_SOURCE_FILES src/bpmpd_interface.cpp)
  set_property(SOURCE src/bpmpd_interface.cpp APPEND PROPERTY COMPILE_DEFINITIONS BPMPD_CALLER="\\\"${CATKIN_DEVEL_PREFIX}/lib/${PROJECT_NAME}/bpmpd_caller\\\"")


  #TODO: Levi check if this is correct.
  set(BPMPD_WORKING_DIR "${CATKIN_DEVEL_PREFIX}/lib/${PROJECT_NAME}/")
  set_property(SOURCE src/bpmpd_caller.cpp APPEND PROPERTY COMPILE_DEFINITIONS BPMPD_WORKING_DIR="${BPMPD_WORKING_DIR}")
  file(COPY src/bpmpd.par DESTINATION ${BPMPD_WORKING_DIR})

  set_property(SOURCE src/solver_interface.cpp APPEND PROPERTY COMPILE_DEFINITIONS HAVE_BPMPD )
endif()

if (GUROBI_FOUND)
  include_directories(${GUROBI_INCLUDE_DIR})
  set_property(SOURCE src/solver_interface.cpp APPEND PROPERTY COMPILE_DEFINITIONS HAVE_GUROBI )
  list(APPEND SCO_SOURCE_FILES src/gurobi_interface.cpp)
endif(GUROBI_FOUND)

add_library(${PROJECT_NAME} ${SCO_SOURCE_FILES})

set (SCO_LINK_LIBS ${catkin_LIBRARIES})
if (GUROBI_FOUND)
  list(APPEND SCO_LINK_LIBS ${GUROBI_LIBRARIES})
endif()
if (HAVE_BPMPD)
  list(APPEND SCO_LINK_LIBS ${BPMPD_LIBRARY})
endif()

target_link_libraries(${PROJECT_NAME} ${SCO_LINK_LIBS})

# Mark executables and/or libraries for installation
install(
  TARGETS ${PROJECT_NAME}
  ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  LIBRARY DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  RUNTIME DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
)

# Mark cpp header files for installation
install(DIRECTORY include/${PROJECT_NAME}/
  DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION}
  FILES_MATCHING PATTERN "*.h" PATTERN "*.hpp"
  PATTERN ".svn" EXCLUDE
 )

if (CATKIN_ENABLE_TESTING)
  set(SCO_TEST_SOURCE
    test/unit.cpp
    test/small-problems-unit.cpp
    test/solver-interface-unit.cpp
  )

  catkin_add_gtest(${PROJECT_NAME}-test ${SCO_TEST_SOURCE})
  target_link_libraries(${PROJECT_NAME}-test ${PROJECT_NAME})
endif()
