### Autonomous Road Inspection Challenge
The challenge theme is to build an autonomous mobile robot software system for a LIMO robot to survey driving area, detect, localise, and quantify road defects (e.g., potholes), and present results on a map.

The repository contains a template ROS package called `KV6022_assessment` will act as a reference point for your developments. You will set it up in your own machine and develop the assignment solution there. The package contains the following items:
   * Navigation parameter file in (`/params/nav2_params.yaml`) 
   * Occupancy grid map in `/maps` folder with 20 mm resolution.
   * An `example_opencv_detector` node
   * `example_waypoint_follower`, `example_nav_through_poses` and `example_nav_to_pose` node

These examples are a good starting point for your assessment. You may choose whether to use the provided example node implementations, however, all nodes you develop or use must be placed in the `KV6022_assessment` package. You can run provided package as follows:

1. Install the Navigation2 packages:
```
sudo apt install ros-humble-navigation2 ros-humble-nav2-bringup ros-humble-tf-transformations
```
2. Set up an alternative DDS (Cyclone DDS or Zenoh). Due to some issues with `Fast DDS` and Navigation in ROS2, it has been recommended to use [Cyclone DDS](https://docs.ros.org/en/humble/Installation/RMW-Implementations/DDS-Implementations/Working-with-Eclipse-CycloneDDS.html) or [Zenoh](https://docs.ros.org/en/humble/Installation/RMW-Implementations/Non-DDS-Implementations/Working-with-Zenoh.html) instead. Install one of the following:
    * Cyclone DDS: `sudo apt install ros-humble-rmw-cyclonedds-cpp`
    * Zenoh: `sudo apt install ros-humble-rmw-zenoh-cpp`
3. Configure ROS 2 to use the chosen DDS. You need to set the `RMW_IMPLEMENTATION` environment variable in your `.bashrc` file. 
    * For switching to Cyclone DDS: `echo export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp >> ~/.bashrc`
    * For Zenoh: `echo export RMW_IMPLEMENTATION=rmw_zenoh_cpp >> ~/.bashrc`

After editing `.bashrc`, close the terminal and open up a new one. When you build and source the package,
* If using `Cyclone DDS`, proceed directly to running the simulation.
* Else using `Zenoh`, start the Zenoh router in a separate terminal: `ros2 run rmw_zenoh_cpp rmw_zenohd`
4. Run the simulation: 
```
ros2 launch limo_gazebosim limo_gazebo_assessment.launch.py
```
5. Run the navigation node (high-resolution maps with adjusted parameters)
```
ros2 launch KV6022_assessment limo_navigation.launch.py
```
6. Run the object detector node: 
```
ros2 run KV6022_assessment example_opencv_detector
```
7. Run the waypoint follower node: 
```
ros2 run KV6022_assessment example_waypoint_follower
```

###  Notes on Navigation

* In case the robot has difficulties in navigating from waypoint to waypoint in case they are too close to obstacles, remember:
1. Check localisation quality: you will need a good position estimate (localisation) of the robot within the given map. If uncertainty is high, use `2D Pose Estimate` in RViz and teleoperate the robot a bit to allow localisation to converge.
2. Tune navigation and costmaps parameters:
Parameters you might adjust:
   * Behaviour profile (aggressive vs cautious):
        * `max_vel_x`, `max_vel_theta`, acceleration limits
   * Safety margins:
        * `robot_radius` (robot footprint)
        * `inflation_radius` (safety bubble around obstacles)
   * Offline changes: Edit [planner.yaml](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/params/planner.yaml) and/or [controller.yaml](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/params/controller.yaml). Increase or decrease costmap inflation radius so the robot does not get too close to walls or obstacles. Changes take effect next time you build. source and launch Nav2.
   * Online changes (dynamic reconfigure): Alternatively, dynamically reconfigure on the fly by opening `ros2 run rqt_gui rqt_gui` or by simply typing `rqt` from the console and then from the menu bar `Plugins/Configuration/Dynamic Reconfigure`. Please select `planner_server`, `local_planner`, `global_costamp` or `local_costamp` in the menu on the left, and and adjust parameters interactively.