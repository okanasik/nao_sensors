cmake_minimum_required(VERSION 2.8)
project(nao_sensors)

# Find catkin macros and libraries, also other catkin packages.
find_package(catkin REQUIRED COMPONENTS
  camera_info_manager
  diagnostic_updater
  driver_base
  dynamic_reconfigure
  image_transport
  nodelet
#  octomap_msgs
  roscpp
  sensor_msgs
  nao_msgs
  tf
  nao_driver
)

find_package(NAOqi QUIET COMPONENTS
    alcommon
    alvision
    alproxies
    alerror
    alvalue
)

# deal with the Python code
catkin_python_setup()

# This driver uses Boost threads
find_package(Boost REQUIRED COMPONENTS thread program_options)

# auto-generate dynamic reconfiguration GUI before catkin_package()
generate_dynamic_reconfigure_options(cfg/NaoCamera.cfg
                                     cfg/NaoMicrophone.cfg)

if( ${NAOqi_FOUND} AND ${Boost_FOUND} )
    message(STATUS "NAOqi C++ SDK found; C++ nodes will be built")
    catkin_package(CATKIN_DEPENDS
                 camera_info_manager
                 diagnostic_updater
                 dynamic_reconfigure
                 image_transport
                 nodelet
                 roscpp
                 tf
               DEPENDS Boost
               LIBRARIES naocamera_nodelet)

    include_directories(
        ${catkin_INCLUDE_DIRS}
        ${Boost_INCLUDE_DIRS}
        ${NAOqi_INCLUDE_DIRS})

    set(NAOCAMERA_SOURCES
        src/nao_camera.cpp
        src/nao_camera_node.cpp
        src/nao_node.cpp)

    add_executable(cameras ${NAOCAMERA_SOURCES})
    target_link_libraries(cameras
        ${catkin_LIBRARIES}
        ${Boost_LIBRARIES}
        ${NAOqi_LIBRARIES})
    add_dependencies(cameras ${catkin_EXPORTED_TARGETS})

    install(TARGETS cameras
        RUNTIME DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION} )

    #### NODELET

    # build and install the nodelet
    add_library(naocamera_nodelet src/nodelet.cpp ${NAOCAMERA_SOURCES})
    add_dependencies(naocamera_nodelet ${catkin_EXPORTED_TARGETS})
    target_link_libraries(naocamera_nodelet
                          ${catkin_LIBRARIES}
                          ${NAOqi_LIBRARIES})
    install(TARGETS naocamera_nodelet
            RUNTIME DESTINATION ${CATKIN_GLOBAL_BIN_DESTINATION}
            ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
            LIBRARY DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION})

    # nodelet launch files
    install(FILES launch/nodelet_manager.launch launch/nodelet_standalone.launch launch/nodelet_pipeline.launch
            DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION})

    ####
else()
    catkin_package()

    # not set as warning to not update the buildfarms
    message(STATUS "-- Cannot find NAOqi C++ SDK! Set NAOqi_DIR to the SDK prefix to build the C++ nodes.")
    message(STATUS "-- The Python nodes will still be installed and can be run provided NAOqi is in your PYTHONPATH.")
endif()

add_subdirectory(./src)

# install sensor scripts
install(PROGRAMS nodes/camera.py
                 nodes/microphone.py
#                 nodes/octomap.py
                 nodes/sonar.py
        DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
)

# install pluginlib definition for the nodelet. In the case the plugin is not compiled
# this is still useful to not display an error from the package.xml parsing
# (the plugin is exported there no matter what)
install(FILES naocamera_nodelet.xml
        DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION})
        
# install launch files
install(DIRECTORY launch/ DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}/launch PATTERN ".svn" EXCLUDE)

# install camera_info yaml
install(DIRECTORY share/ DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}/share PATTERN ".svn" EXCLUDE)

