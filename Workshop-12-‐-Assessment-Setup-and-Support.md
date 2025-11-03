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