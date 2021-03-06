cmake_minimum_required(VERSION 2.8.3)
project(ira_photonfocus_driver)

#set(CMAKE_BUILD_TYPE Release)
set(CMAKE_BUILD_TYPE Debug)
SET (CMAKE_CXX_FLAGS "-std=c++11")

# Architecture detection borrowed from: https://github.com/petroules/solar-cmake/blob/master/TargetArch.cmake
# Based on the Qt 5 processor detection code, so should be very accurate
# https://qt.gitorious.org/qt/qtbase/blobs/master/src/corelib/global/qprocessordetection.h
# Currently handles arm (v5, v6, v7), x86 (32/64), ia64, and ppc (32/64)

# Regarding POWER/PowerPC, just as is noted in the Qt source,
# "There are many more known variants/revisions that we do not handle/detect."

set(archdetect_c_code "
#if defined(__arm__) || defined(__TARGET_ARCH_ARM)
#if defined(__ARM_ARCH_7__) \\
|| defined(__ARM_ARCH_7A__) \\
|| defined(__ARM_ARCH_7R__) \\
|| defined(__ARM_ARCH_7M__) \\
|| (defined(__TARGET_ARCH_ARM) && __TARGET_ARCH_ARM-0 >= 7)
#error cmake_ARCH armv7
#elif defined(__ARM_ARCH_6__) \\
|| defined(__ARM_ARCH_6J__) \\
|| defined(__ARM_ARCH_6T2__) \\
|| defined(__ARM_ARCH_6Z__) \\
|| defined(__ARM_ARCH_6K__) \\
|| defined(__ARM_ARCH_6ZK__) \\
|| defined(__ARM_ARCH_6M__) \\
|| (defined(__TARGET_ARCH_ARM) && __TARGET_ARCH_ARM-0 >= 6)
#error cmake_ARCH armv6
#elif defined(__ARM_ARCH_5TEJ__) \\
|| (defined(__TARGET_ARCH_ARM) && __TARGET_ARCH_ARM-0 >= 5)
#error cmake_ARCH armv5
#else
#error cmake_ARCH arm
#endif
#elif defined(__i386) || defined(__i386__) || defined(_M_IX86)
#error cmake_ARCH i386
#elif defined(__x86_64) || defined(__x86_64__) || defined(__amd64) || defined(_M_X64)
#error cmake_ARCH x86_64
#elif defined(__ia64) || defined(__ia64__) || defined(_M_IA64)
#error cmake_ARCH ia64
#elif defined(__ppc__) || defined(__ppc) || defined(__powerpc__) \\
|| defined(_ARCH_COM) || defined(_ARCH_PWR) || defined(_ARCH_PPC) \\
|| defined(_M_MPPC) || defined(_M_PPC)
#if defined(__ppc64__) || defined(__powerpc64__) || defined(__64BIT__)
#error cmake_ARCH ppc64
#else
#error cmake_ARCH ppc
#endif
#endif
#error cmake_ARCH unknown
")

# Set ppc_support to TRUE before including this file or ppc and ppc64
# will be treated as invalid architectures since they are no longer supported by Apple

function(target_architecture output_var)
    if(APPLE AND CMAKE_OSX_ARCHITECTURES)
        # On OS X we use CMAKE_OSX_ARCHITECTURES *if* it was set
        # First let's normalize the order of the values

        # Note that it's not possible to compile PowerPC applications if you are using
        # the OS X SDK version 10.6 or later - you'll need 10.4/10.5 for that, so we
        # disable it by default
        # See this page for more information:
        # http://stackoverflow.com/questions/5333490/how-can-we-restore-ppc-ppc64-as-well-as-full-10-4-10-5-sdk-support-to-xcode-4

        # Architecture defaults to i386 or ppc on OS X 10.5 and earlier, depending on the CPU type detected at runtime.
        # On OS X 10.6+ the default is x86_64 if the CPU supports it, i386 otherwise.

        foreach(osx_arch ${CMAKE_OSX_ARCHITECTURES})
            if("${osx_arch}" STREQUAL "ppc" AND ppc_support)
                set(osx_arch_ppc TRUE)
            elseif("${osx_arch}" STREQUAL "i386")
                set(osx_arch_i386 TRUE)
            elseif("${osx_arch}" STREQUAL "x86_64")
                set(osx_arch_x86_64 TRUE)
            elseif("${osx_arch}" STREQUAL "ppc64" AND ppc_support)
                set(osx_arch_ppc64 TRUE)
            else()
                message(FATAL_ERROR "Invalid OS X arch name: ${osx_arch}")
            endif()
        endforeach()

        # Now add all the architectures in our normalized order
        if(osx_arch_ppc)
            list(APPEND ARCH ppc)
        endif()

        if(osx_arch_i386)
            list(APPEND ARCH i386)
        endif()

        if(osx_arch_x86_64)
            list(APPEND ARCH x86_64)
        endif()

        if(osx_arch_ppc64)
            list(APPEND ARCH ppc64)
        endif()
    else()
        file(WRITE "${CMAKE_BINARY_DIR}/arch.c" "${archdetect_c_code}")

        enable_language(C)

        # Detect the architecture in a rather creative way...
        # This compiles a small C program which is a series of ifdefs that selects a
        # particular #error preprocessor directive whose message string contains the
        # target architecture. The program will always fail to compile (both because
        # file is not a valid C program, and obviously because of the presence of the
        # #error preprocessor directives... but by exploiting the preprocessor in this
        # way, we can detect the correct target architecture even when cross-compiling,
        # since the program itself never needs to be run (only the compiler/preprocessor)
        try_run(
            run_result_unused
            compile_result_unused
            "${CMAKE_BINARY_DIR}"
            "${CMAKE_BINARY_DIR}/arch.c"
            COMPILE_OUTPUT_VARIABLE ARCH
            CMAKE_FLAGS CMAKE_OSX_ARCHITECTURES=${CMAKE_OSX_ARCHITECTURES}
        )

        # Parse the architecture name from the compiler output
        string(REGEX MATCH "cmake_ARCH ([a-zA-Z0-9_]+)" ARCH "${ARCH}")

        # Get rid of the value marker leaving just the architecture name
        string(REPLACE "cmake_ARCH " "" ARCH "${ARCH}")

        # If we are compiling with an unknown architecture this variable should
        # already be set to "unknown" but in the case that it's empty (i.e. due
        # to a typo in the code), then set it to unknown
        if (NOT ARCH)
            set(ARCH unknown)
        endif()
    endif()

    set(${output_var} "${ARCH}" PARENT_SCOPE)
endfunction()

# architecture:
target_architecture(ARCH)

message(STATUS "${ARCH}")

## Find catkin macros and libraries
## if COMPONENTS list like find_package(catkin REQUIRED COMPONENTS xyz)
## is used, also find other catkin packages
find_package(catkin REQUIRED COMPONENTS
  roscpp
  cv_bridge
  image_transport
  dynamic_reconfigure
  driver_base
  camera_info_manager
)

find_package(OpenCV REQUIRED)
find_package(Boost REQUIRED COMPONENTS thread)

# EBUS SDK
set(EBUS_INCLUDE_DIRS ebus_sdk/include)
set(EBUS_LIBRARIES PvBase PvDevice PvPersistence PvGenICam PvStream PvTransmitter PvVirtualDevice PvAppUtils PvSerial PvGUI PvBuffer)

if(${ARCH} STREQUAL "x86_64")
    set(EBUS_LIB_DIRS ${PROJECT_SOURCE_DIR}/ebus_sdk/lib_x86_64)
    link_directories(${EBUS_LIB_DIRS})
    link_directories("${EBUS_LIB_DIRS}/genicam/bin/Linux64_x64")
    link_directories("${EBUS_LIB_DIRS}/genicam/bin/Linux64_x64/GenApi/Generic")
else()
    set(EBUS_LIB_DIRS ${PROJECT_SOURCE_DIR}/ebus_sdk/lib_i686)
    link_directories(${EBUS_LIB_DIRS})
    link_directories("${EBUS_LIB_DIRS}/genicam/bin/Linux32_i86")
    link_directories("${EBUS_LIB_DIRS}/genicam/bin/Linux32_i86/GenApi/Generic")
endif()

add_definitions(-D_UNIX_ -D_LINUX_)

include(FindGnuTLS)

if(NOT GNUTLS_FOUND)

    message("GNUTLS not found! Aborting...")

    fail()

endif(NOT GNUTLS_FOUND)

## System dependencies are found with CMake's conventions
# find_package(Boost REQUIRED COMPONENTS system)

## Uncomment this if the package has a setup.py. This macro ensures
## modules and global scripts declared therein get installed
## See http://ros.org/doc/api/catkin/html/user_guide/setup_dot_py.html
# catkin_python_setup()

################################################
## Declare ROS messages, services and actions ##
################################################

## To declare and build messages, services or actions from within this
## package, follow these steps:
## * Let MSG_DEP_SET be the set of packages whose message types you use in
##   your messages/services/actions (e.g. std_msgs, actionlib_msgs, ...).
## * In the file package.xml:
##   * add a build_depend and a run_depend tag for each package in MSG_DEP_SET
##   * If MSG_DEP_SET isn't empty the following dependencies might have been
##     pulled in transitively but can be declared for certainty nonetheless:
##     * add a build_depend tag for "message_generation"
##     * add a run_depend tag for "message_runtime"
## * In this file (CMakeLists.txt):
##   * add "message_generation" and every package in MSG_DEP_SET to
##     find_package(catkin REQUIRED COMPONENTS ...)
##   * add "message_runtime" and every package in MSG_DEP_SET to
##     catkin_package(CATKIN_DEPENDS ...)
##   * uncomment the add_*_files sections below as needed
##     and list every .msg/.srv/.action file to be processed
##   * uncomment the generate_messages entry below
##   * add every package in MSG_DEP_SET to generate_messages(DEPENDENCIES ...)

## Generate messages in the 'msg' folder
# add_message_files(
#   FILES
#   Message1.msg
#   Message2.msg
# )

## Generate services in the 'srv' folder
# add_service_files(
#   FILES
#   Service1.srv
#   Service2.srv
# )

## Generate actions in the 'action' folder
# add_action_files(
#   FILES
#   Action1.action
#   Action2.action
# )

## Generate added messages and services with any dependencies listed here
# generate_messages(
#   DEPENDENCIES
#   std_msgs  # Or other packages containing msgs
# )

generate_dynamic_reconfigure_options(
  cfg/photonfocus.cfg
)

###################################
## catkin specific configuration ##
###################################
## The catkin_package macro generates cmake config files for your package
## Declare things to be passed to dependent projects
## INCLUDE_DIRS: uncomment this if you package contains header files
## LIBRARIES: libraries you create in this project that dependent projects also need
## CATKIN_DEPENDS: catkin_packages dependent projects also need
## DEPENDS: system dependencies of this project that dependent projects also need
catkin_package(
#  INCLUDE_DIRS include
#  LIBRARIES photonfocus_camera
#  CATKIN_DEPENDS cv_bridge image_transport roscpp
#  DEPENDS system_lib
)

###########
## Build ##
###########

## Specify additional locations of header files
## Your package locations should be listed before other locations
# include_directories(include)
include_directories(
  include
  ${catkin_INCLUDE_DIRS}
  ${EBUS_INCLUDE_DIRS}
  ${GNUTLS_INCLUDE_DIR}
)

## Declare a cpp library
# add_library(photonfocus_camera
#   src/${PROJECT_NAME}/photonfocus_camera.cpp
# )
add_library(ira_photonfocus src/photonfocus_camera.cpp)

## Declare a cpp executable
# add_executable(photonfocus_camera_node src/photonfocus_camera_node.cpp)

# DRIVER
add_executable(ira_photonfocus_driver_mono src/photonfocus_driver_mono.cpp)
add_executable(ira_photonfocus_driver_stereo src/photonfocus_driver_stereo.cpp)

# UTILITIES
add_executable(set_ip_photonfocus src/utilities/set_ip.cpp)
add_executable(find_camera_photonfocus src/utilities/find_camera.cpp)

## Add cmake target dependencies of the executable/library
## as an example, message headers may need to be generated before nodes
# add_dependencies(photonfocus_camera_node photonfocus_camera_generate_messages_cpp)
add_dependencies(ira_photonfocus_driver_mono ${PROJECT_NAME}_gencfg)
add_dependencies(ira_photonfocus_driver_stereo ${PROJECT_NAME}_gencfg)


## Specify libraries to link a library or executable target against
# target_link_libraries(photonfocus_camera_node
#   ${catkin_LIBRARIES}
# )
target_link_libraries(ira_photonfocus
    ${EBUS_LIBRARIES}
    ${OpenCV_LIBRARIES}
    ${Boost_LIBRARIES}
    ${GNUTLS_LIBRARIES}
)

target_link_libraries(ira_photonfocus_driver_mono
    ira_photonfocus
    ${EBUS_LIBRARIES}
    ${catkin_LIBRARIES}
    ${GNUTLS_LIBRARIES}
)

target_link_libraries(ira_photonfocus_driver_stereo
    ira_photonfocus
    ${EBUS_LIBRARIES}
    ${catkin_LIBRARIES}
    ${GNUTLS_LIBRARIES}
)

target_link_libraries(set_ip_photonfocus
    ${EBUS_LIBRARIES}
    ${GNUTLS_LIBRARIES}
)

target_link_libraries(find_camera_photonfocus
    ${EBUS_LIBRARIES}
    ${GNUTLS_LIBRARIES}
)

#############
## Install ##
#############

# all install targets should use catkin DESTINATION variables
# See http://ros.org/doc/api/catkin/html/adv_user_guide/variables.html

## Mark executable scripts (Python etc.) for installation
## in contrast to setup.py, you can choose the destination
# install(PROGRAMS
#   scripts/my_python_script
#   DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
# )

## Mark executables and/or libraries for installation
# install(TARGETS photonfocus_camera photonfocus_camera_node
#   ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
#   LIBRARY DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
#   RUNTIME DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
# )

## Mark cpp header files for installation
# install(DIRECTORY include/${PROJECT_NAME}/
#   DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION}
#   FILES_MATCHING PATTERN "*.h"
#   PATTERN ".svn" EXCLUDE
# )

## Mark other files for installation (e.g. launch and bag files, etc.)
# install(FILES
#   # myfile1
#   # myfile2
#   DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}
# )

#############
## Testing ##
#############

## Add gtest based cpp test target and link libraries
# catkin_add_gtest(${PROJECT_NAME}-test test/test_photonfocus_camera.cpp)
# if(TARGET ${PROJECT_NAME}-test)
#   target_link_libraries(${PROJECT_NAME}-test ${PROJECT_NAME})
# endif()

## Add folders to be run by python nosetests
# catkin_add_nosetests(test)
