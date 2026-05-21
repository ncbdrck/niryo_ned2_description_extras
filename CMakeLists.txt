cmake_minimum_required(VERSION 3.0.2)
project(niryo_ned2_description_extras)

find_package(catkin REQUIRED)

catkin_package()

install(DIRECTORY urdf launch config
  DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}
)
