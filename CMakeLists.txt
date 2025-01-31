cmake_minimum_required(VERSION 3.5)
project(slam_toolbox)

if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 17)
endif()
cmake_policy(SET CMP0077 NEW)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_LIST_DIR}/CMake/")
list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_LIST_DIR}/lib/karto_sdk/cmake)

find_package(ament_cmake REQUIRED)
find_package(message_filters REQUIRED)
find_package(nav_msgs REQUIRED)
find_package(rclcpp REQUIRED)
find_package(sensor_msgs REQUIRED)
find_package(tf2 REQUIRED)
find_package(tf2_ros REQUIRED)
find_package(visualization_msgs REQUIRED)
find_package(pluginlib REQUIRED)
find_package(tf2_geometry_msgs REQUIRED)
find_package(tf2_sensor_msgs REQUIRED)
find_package(std_msgs REQUIRED)
find_package(std_srvs REQUIRED)
find_package(builtin_interfaces REQUIRED)
find_package(rosidl_default_generators REQUIRED)
find_package(interactive_markers REQUIRED)

find_package(PkgConfig REQUIRED)
find_package(Eigen3 REQUIRED)
find_package(CHOLMOD REQUIRED)

if(BUILD_RVIZ_PLUGIN)
    find_package(rviz_common REQUIRED)
    find_package(rviz_default_plugins REQUIRED)
    find_package(rviz_ogre_vendor REQUIRED)
    find_package(rviz_rendering REQUIRED)
    find_package(Qt5 REQUIRED COMPONENTS Core Gui Widgets Test Concurrent)
endif(BUILD_RVIZ_PLUGIN)

#karto_sdk lib
set(BUILD_SHARED_LIBS ON)
add_subdirectory(lib/karto_sdk)

set(CMAKE_INCLUDE_CURRENT_DIR ON)

set(dependencies
  rclcpp
  message_filters
  nav_msgs
  sensor_msgs
  tf2
  tf2_ros
  visualization_msgs
  pluginlib
  tf2_geometry_msgs
  tf2_sensor_msgs
  std_msgs
  std_srvs
  builtin_interfaces
  interactive_markers
)

if(BUILD_RVIZ_PLUGIN)
    list(APPEND dependencies
        rviz_common
        rviz_default_plugins
        rviz_ogre_vendor
        rviz_rendering
        Qt5
    )
endif(BUILD_RVIZ_PLUGIN)

set(libraries
    toolbox_common
    async_slam_toolbox
    sync_slam_toolbox
    localization_slam_toolbox
    lifelong_slam_toolbox
    map_and_localization_slam_toolbox
)

if(BUILD_RVIZ_PLUGIN)
    list(APPEND libraries SlamToolboxPlugin)
endif(BUILD_RVIZ_PLUGIN)

if(BUILD_CERES_PLUGIN)
    list(APPEND libraries ceres_solver_plugin)
    find_package(Ceres REQUIRED COMPONENTS SuiteSparse)
endif(BUILD_CERES_PLUGIN)

set(CMAKE_INCLUDE_CURRENT_DIR ON)
add_definitions(-DQT_NO_KEYWORDS)
find_package(Boost REQUIRED system serialization filesystem thread)

include_directories(include lib/karto_sdk/include
                            ${EIGEN3_INCLUDE_DIRS}
                            ${CHOLMOD_INCLUDE_DIR}
                            ${Boost_INCLUDE_DIRS}
)

if(BUILD_CERES_PLUGIN)
    include_directories(${CERES_INCLUDES})
endif(BUILD_CERES_PLUGIN)

add_definitions(${EIGEN3_DEFINITIONS})

rosidl_generate_interfaces(${PROJECT_NAME}
  srvs/Pause.srv
  srvs/ClearQueue.srv
  srvs/ToggleInteractive.srv
  srvs/Clear.srv
  srvs/SaveMap.srv
  srvs/LoopClosure.srv
  srvs/MergeMaps.srv
  srvs/AddSubmap.srv
  srvs/DeserializePoseGraph.srv
  srvs/SerializePoseGraph.srv
  DEPENDENCIES builtin_interfaces geometry_msgs std_msgs nav_msgs visualization_msgs
)

##### rviz Plugin
if(BUILD_RVIZ_PLUGIN)
    qt5_wrap_cpp(MOC_FILES rviz_plugin/slam_toolbox_rviz_plugin.hpp)
    add_library(SlamToolboxPlugin SHARED
      rviz_plugin/slam_toolbox_rviz_plugin.cpp
      ${MOC_FILES})
    ament_target_dependencies(SlamToolboxPlugin
     ${dependencies}
    )
    target_include_directories(SlamToolboxPlugin PUBLIC
      ${Qt5Widgets_INCLUDE_DIRS}
      ${OGRE_INCLUDE_DIRS}
    )
    target_link_libraries(SlamToolboxPlugin ${QT_LIBRARIES} rviz_common::rviz_common)
    rosidl_get_typesupport_target(cpp_typesupport_target ${PROJECT_NAME} "rosidl_typesupport_cpp")
    target_link_libraries(SlamToolboxPlugin "${cpp_typesupport_target}")
    target_compile_definitions(SlamToolboxPlugin PUBLIC "PLUGINLIB__DISABLE_BOOST_FUNCTIONS")
    target_compile_definitions(SlamToolboxPlugin PRIVATE "RVIZ_DEFAULT_PLUGINS_BUILDING_LIBRARY")
    pluginlib_export_plugin_description_file(rviz_common rviz_plugins.xml) 
endif(BUILD_RVIZ_PLUGIN)

##### Ceres Plugin
if(BUILD_CERES_PLUGIN)
    add_library(ceres_solver_plugin solvers/ceres_solver.cpp)
    ament_target_dependencies(ceres_solver_plugin ${dependencies})
    target_link_libraries(ceres_solver_plugin ${CERES_LIBRARIES}
                                              ${Boost_LIBRARIES}
                                              kartoSlamToolbox
    )
    rosidl_get_typesupport_target(cpp_typesupport_target ${PROJECT_NAME} "rosidl_typesupport_cpp")
    target_link_libraries(ceres_solver_plugin "${cpp_typesupport_target}")
    pluginlib_export_plugin_description_file(slam_toolbox solver_plugins.xml)
endif(BUILD_CERES_PLUGIN)

#### Tool lib for mapping
add_library(toolbox_common src/slam_toolbox_common.cpp src/map_saver.cpp src/loop_closure_assistant.cpp src/laser_utils.cpp src/slam_mapper.cpp)
ament_target_dependencies(toolbox_common
  ${dependencies}
)
target_link_libraries(toolbox_common kartoSlamToolbox ${Boost_LIBRARIES})
rosidl_get_typesupport_target(cpp_typesupport_target ${PROJECT_NAME} "rosidl_typesupport_cpp")
target_link_libraries(toolbox_common "${cpp_typesupport_target}")

#### Mapping executibles
add_library(async_slam_toolbox src/slam_toolbox_async.cpp)
target_link_libraries(async_slam_toolbox toolbox_common kartoSlamToolbox ${Boost_LIBRARIES})
add_executable(async_slam_toolbox_node src/slam_toolbox_async_node.cpp)
target_link_libraries(async_slam_toolbox_node async_slam_toolbox)

add_library(sync_slam_toolbox src/slam_toolbox_sync.cpp)
target_link_libraries(sync_slam_toolbox toolbox_common kartoSlamToolbox ${Boost_LIBRARIES})
add_executable(sync_slam_toolbox_node src/slam_toolbox_sync_node.cpp)
target_link_libraries(sync_slam_toolbox_node sync_slam_toolbox)

add_library(localization_slam_toolbox src/slam_toolbox_localization.cpp)
target_link_libraries(localization_slam_toolbox toolbox_common kartoSlamToolbox ${Boost_LIBRARIES})
add_executable(localization_slam_toolbox_node src/slam_toolbox_localization_node.cpp)
target_link_libraries(localization_slam_toolbox_node localization_slam_toolbox)

add_library(lifelong_slam_toolbox src/experimental/slam_toolbox_lifelong.cpp)
target_link_libraries(lifelong_slam_toolbox toolbox_common kartoSlamToolbox ${Boost_LIBRARIES})
add_executable(lifelong_slam_toolbox_node src/experimental/slam_toolbox_lifelong_node.cpp)
target_link_libraries(lifelong_slam_toolbox_node lifelong_slam_toolbox)

add_library(map_and_localization_slam_toolbox src/experimental/slam_toolbox_map_and_localization.cpp)
target_link_libraries(map_and_localization_slam_toolbox localization_slam_toolbox toolbox_common kartoSlamToolbox ${Boost_LIBRARIES})
add_executable(map_and_localization_slam_toolbox_node src/experimental/slam_toolbox_map_and_localization_node.cpp)
target_link_libraries(map_and_localization_slam_toolbox_node map_and_localization_slam_toolbox)

#### Merging maps tool
add_executable(merge_maps_kinematic src/merge_maps_kinematic.cpp)
target_link_libraries(merge_maps_kinematic kartoSlamToolbox toolbox_common)

#### Install
install(TARGETS async_slam_toolbox_node
                sync_slam_toolbox_node
                localization_slam_toolbox_node
                map_and_localization_slam_toolbox_node
                merge_maps_kinematic
                ${libraries}
                kartoSlamToolbox
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION lib/${PROJECT_NAME}
)

if(BUILD_RVIZ_PLUGIN)
    install(TARGETS SlamToolboxPlugin
      EXPORT SlamToolboxPlugin
      ARCHIVE DESTINATION lib
      LIBRARY DESTINATION lib
      RUNTIME DESTINATION bin
      INCLUDES DESTINATION include
    )
endif(BUILD_RVIZ_PLUGIN)

install(DIRECTORY include/
  DESTINATION include
)

install(DIRECTORY launch
  DESTINATION share/${PROJECT_NAME}
  PATTERN "lifelong_launch.py" EXCLUDE
)

install(DIRECTORY config
  DESTINATION share/${PROJECT_NAME}
)

install(FILES solver_plugins.xml rviz_plugins.xml
  DESTINATION share
)

ament_export_include_directories(include)
ament_export_libraries(${libraries} kartoSlamToolbox)
ament_export_dependencies(${dependencies})
if(BUILD_RVIZ_PLUGIN)
    ament_export_targets(SlamToolboxPlugin HAS_LIBRARY_TARGET)
endif(BUILD_RVIZ_PLUGIN)
ament_package()
