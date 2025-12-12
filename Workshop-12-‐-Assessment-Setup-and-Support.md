
### 🚧 Autonomous Road Inspection Challenge
The challenge theme is to build an autonomous mobile robot software system for a LIMO robot to survey driving area, detect, localise, and quantify road defects (e.g., potholes), and present results on a map.

<p align="center">
<img width="800" height="480" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/assessment.png" />
</p>



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
> [!WARNING]
> **DDS / RMW choice for Nav2**
>
> By default, ROS 2 on Humble typically uses **Fast DDS** as the middleware
> (RMW implementation). In many cases this works fine, but it is known that
> Fast DDS can sometimes cause issues with **Nav2** 
>
> If you see missing /map, /tf, costmap topics, or Nav2 nodes not reacting, consider switching to [Cyclone DDS](https://docs.ros.org/en/humble/Installation/RMW-Implementations/DDS-Implementations/Working-with-Eclipse-CycloneDDS.html) or [Zenoh](https://docs.ros.org/en/humble/Installation/RMW-Implementations/Non-DDS-Implementations/Working-with-Zenoh.html).  
> Install one of the following:
>    * Cyclone DDS: `sudo apt install ros-humble-rmw-cyclonedds-cpp`
>    * Zenoh: `sudo apt install ros-humble-rmw-zenoh-cpp`
>
> Configure ROS 2 to use the chosen DDS. You need to set the `RMW_IMPLEMENTATION` environment variable in your `.bashrc` file. 
>    * For switching to Cyclone DDS: `echo export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp >> ~/.bashrc`
>    * For Zenoh: `echo export RMW_IMPLEMENTATION=rmw_zenoh_cpp >> ~/.bashrc`
>
> After editing `.bashrc`, close the terminal and open up a new one. When you build and source the package,
>    * If using `Cyclone DDS`, proceed directly to running the simulation.
>    * Else using `Zenoh`, start the Zenoh router in a separate terminal: `ros2 run rmw_zenoh_cpp rmw_zenohd`

2. Run the simulation: 
```
ros2 launch limo_gazebosim limo_gazebo_assessment.launch.py
```
3. Run the navigation node (high-resolution maps with adjusted parameters)
```
ros2 launch KV6022_assessment limo_navigation.launch.py
```
4. Run the object detector node: 
```
ros2 run KV6022_assessment example_opencv_detector
```
5. Run the waypoint follower node: 
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

### Object detection and localisation in a map

The example node [object_localisation.py](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/example_codes/object_localisation.py) demonstrates how to detect colour objects in images and then perform image -> camera -> global coordinates transformations. The node subscribes to colour and depth images and utilises the camera_info topic which includes camera geometry information (focal point, resolution, etc.). The colour detector is very similar to the basic one used in [Workshop 5](https://github.com/kivrakh/KV6022_limo_ros2/wiki/Workshop-5-%E2%80%90-Robot-Perception-and-Vision). The additional functionality includes transforming the image coordinates to a camera frame and subsequently from a camera frame to a global frame which is `map` in our case. Observe the calculated object location in relation to map.

> [!WARNING]
> If you encounter any of the following errors while running the node, try re-running the node:
> * `AttributeError: 'NoneType' object has no attribute 'encoding'`
> * `tf2.ExtrapolationException: Lookup would require extrapolation at time 1498.045000, but only time 1498.929000 is in the buffer, when looking up transform from frame [depth_link] to frame [map]`


### Object counting in 3D

The example [object_counter.py](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/example_codes/object_counter.py) demonstrates how to count the detected objects in global coordinates with a simple filter preventing double counting. The node subscribes to `object_location` topic and keeps track of all detected objects. The new detection is first checked for its distance to all counted objects so far and if it is detected close to the existing object (distance below `detection_threshold`), then it is ignored. This allows the robot to detect the objects from multiple viewpoints without registering multiple counts.
   * To see how the counter works, launch the simulator.
   * Run the detector node
   * Run the counter node. The node prints out a full list of all objects in the terminal.
   * The counter's key parameter is `detection_threshold`. Set that to different values and note its behaviour with different robot speeds and object sizes.