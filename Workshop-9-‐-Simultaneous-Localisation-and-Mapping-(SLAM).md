### Overview
In this workshop, you will:
* Build a 2D occupancy grid map of the environment using SLAM
* Use the `slam_toolbox` package with the LIMO robot's laser scanner
* Save the generated map to reuse it later for navigation
* Explore how environment changes and parameter tuning affect mapping quality

### Preparation 1:
To build the map we will use the `async_slam_toolbox_node` node from the [slam_toolbox](https://github.com/SteveMacenski/slam_toolbox) package, as it is the default offering for [Nav2](https://docs.nav2.org/) and is very well maintained. More information can be found in the [slides to a ROSCon 2019 talk](https://roscon.ros.org/2019/talks/roscon2019_slamtoolbox.pdf) (or go watch it [on Vimeo](https://vimeo.com/378682207)).

This node subscribes to the `/scan` topic to obtain data about the surrounding environment. In addition, it subscribes to `/tf` messages to obtain the position of the laser scanner and the robot relative to the starting point. The algorithm can then determine the distance of the robot from the surrounding obstacles and on this basis create an 2D occupation/occupancy map. This map is published in topic `/map` in message `nav_msgs/OccupancyGrid`.

1. Lets start our work from installing `slam_toolbox` package.
```
sudo apt install ros-humble-slam-toolbox
```
To run this node it is necessary to set parameters. There are many parameters provided by `async_slam_toolbox_node` node. All the parameters are in the [slam_toolbox github repository](https://github.com/SteveMacenski/slam_toolbox#toolbox-params), and the most important ones are:

* `base_frame` - name of frame related with robot, in our case it will be base_link,
* `odom_frame` - name of frame related with starting point, in our case it will be odom,
* `scan_topic` - name of topic with sensor data, in our case it will be /scan_filtered,
* `resolution` - resolution of the map in meters, in our case it will be set to 0.04,
* `max_laser_range` - the maximum usable range of the laser, in our case it will be 12.0 meters
* `mode` - mapping or localization mode for performance optimizations in the Ceres problem creation

We will use a prepared below configuration file [slam.yaml](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/params/slam.yaml). If you completed [Week 8 workshop](https://github.com/kivrakh/KV6022_limo_ros2/wiki/Workshop-8-%E2%80%90-Map%E2%80%90based-Localization), you can continue using same package (`limo_localisation`) and add `slam.yaml` to `limo_localisation/params/` folder. If you have not completed Week 8, please refer back to that workshop first and create the `limo_localisation` package, then return here and add `slam.yaml` to its `params` folder.
<pre>
slam:
  ros__parameters:

    # Plugin params
    solver_plugin: solver_plugins::CeresSolver
    ceres_linear_solver: SPARSE_NORMAL_CHOLESKY
    ceres_preconditioner: SCHUR_JACOBI
    ceres_trust_strategy: LEVENBERG_MARQUARDT
    ceres_dogleg_type: TRADITIONAL_DOGLEG
    ceres_loss_function: None

    # ROS Parameters
    <mark>odom_frame: odom</mark>
    map_frame: map
    <mark>base_frame: base_link</mark>
    <mark>scan_topic: /scan</mark>
    <mark># mode: mapping # It will be set in launch file</mark>
    debug_logging: false
    throttle_scans: 1
    transform_publish_period: 0.02 # If 0 never publishes odometry
    map_update_interval: 2.0
    <mark>resolution: 0.05</mark>
    <mark>max_laser_range: 12.0</mark>
    minimum_time_interval: 0.1
    transform_timeout: 0.2
    tf_buffer_duration: 20.0
    stack_size_to_use: 40000000
    enable_interactive_mode: false

    # General Parameters
    use_scan_matching: true
    use_scan_barycenter: true
    minimum_travel_distance: 0.3
    minimum_travel_heading: 0.5
    scan_buffer_size: 10
    scan_buffer_maximum_scan_distance: 7.0
    link_match_minimum_response_fine: 0.1
    link_scan_maximum_distance: 1.0
    loop_search_maximum_distance: 3.0
    do_loop_closing: true
    loop_match_minimum_chain_size: 10
    loop_match_maximum_variance_coarse: 3.0
    loop_match_minimum_response_coarse: 0.35
    loop_match_minimum_response_fine: 0.45

    # Correlation Parameters - Correlation Parameters
    correlation_search_space_dimension: 0.5
    correlation_search_space_resolution: 0.01
    correlation_search_space_smear_deviation: 0.1

    # Correlation Parameters - Loop Closure Parameters
    loop_search_space_dimension: 8.0
    loop_search_space_resolution: 0.05
    loop_search_space_smear_deviation: 0.03

    # Scan Matcher Parameters
    distance_variance_penalty: 0.5
    angle_variance_penalty: 1.0
    fine_search_angle_offset: 0.00349
    coarse_search_angle_offset: 0.349
    coarse_angle_resolution: 0.0349
    minimum_angle_penalty: 0.9
    minimum_distance_penalty: 0.5
    use_response_expansion: true
</pre>

### Preparation 2 – Launch File for `slam_toolbox`
We now create a launch file to start `async_slam_toolbox_node` with our `slam.yaml` configuration.
1. In your existing package (e.g. `limo_localisation`), create the file: `launch/limo_slam.launch.py`
2. Use the following content (adjust `pkg_name` if your package name is different):
<pre>
import os
from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch_ros.actions import Node, SetParameter

def generate_launch_description():
    ld = LaunchDescription()

    <mark>pkg_name = 'limo_localisation'</mark>
    slam_params_file = os.path.join(get_package_share_directory(pkg_name),'params','slam.yaml')
   
    slam_node = Node(
        package='slam_toolbox',
        executable='async_slam_toolbox_node',
        name='slam',
        parameters=[slam_params_file],
    )

    
    # Add actions to LaunchDescription
    ld.add_action(SetParameter(name='use_sim_time', value=True))
    ld.add_action(slam_node)

    return ld
</pre>

Make sure:
* `slam.yaml` is stored under your package's `params` folder.
* `setup.py` installs `params` properly (you already did this in the localisation workshop).

Rebuild and source your workspace:
```
colcon build
source install/setup.bash
```
### Task 1: Mapping
We are ready to build a map of the environment using `slam_toolbox` while driving the robot around.
* Launch the Gazebo simulator 
```
ros2 launch limo_gazebosim limo_gazebo_diff.launch.py.
```
* Launch the slam toolbox
```
ros2 launch limo_localisation limo_slam.launch.py
```
* Make sure SLAM provides the `map`→`odom` transform (`ros2 run rqt_tf_tree rqt_tf_tree`) and outputs `/map` topic (`ros2 topic list`). 
* To visualise the SLAM process. Launch RViz `rviz2`
  * Set Fixed Frame to `map`.
  * Add the topics you want to visualize such as: `/scan` and `/map` topics. You can select them from `By topic` tab. You may also add `Robot Model` plugin so you can see the robot.
  * Initially, you should see a small portion of the map (whatever the laser currently observes).
* Initializing a node can take a while, so you may need to wait few second until you see generated map. Starting state should be similar to below picture:

<p align="center">
<img width="800" height="400" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/slam_start.png" />
</p>

* Now slowly drive your robot (`ros2 run teleop_twist_keyboard teleop_twist_keyboard`) around and observe as new parts of map are added, continue until all places are explored. Final map should look like below:

<p align="center">
<img width="594" height="570" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/slam_end.png" />
</p>
* Check how well the laser scans overlap with the generated map. Note that:

   * Very fast movements or
   * Large pure rotations on the spot

can lead to distorted or inconsistent maps (poor SLAM performance)
* In this way, a occupancy map was created, where white points indicate free space, black points are a wall or other obstacle, and transparently marked places not yet visited. 

### Task 2: Saving the map
* The `/map` topic is available till node is running, when you close your node you will lose all your explored map. To reuse the map (e.g. later for AMCL / Nav2), you need to save it using `nav2_map_server`.

* Save map executing below command:
```
ros2 run nav2_map_server map_saver_cli -f map
```
This will create two files (`map.yaml` & `map.pgm`) in the current directory. Inspect the created files (you do not need to fully understand the contents yet, but you should recognise the structure: image + resolution + origin + thresholds) and move these files into your package's `maps` folder, e.g.: 
```
limo_localisation/
  └── maps/
    ├── map.pgm
    └── map.yaml
```
### Task 3: Changing the `slam_toolbox` parameters:

`slam_toolbox` node parameters are set by passing parameters to the node in the launch file via a `slam.yaml` file. 

* Firstly, read about the slam\_toolbox [key parameters](https://github.com/SteveMacenski/slam_toolbox#toolbox-params).
* Make changes to some key parameters in the `slam.yaml` yaml file:
    * Increase the map `resolution` to `0.02`.
    * Adjust how often the map is updated: `map_update_interval` to 1.0
* Rebuild and source your workspace, then relaunch SLAM: 
```
ros2 launch limo_localisation limo_slam.launch.py
```
* Does a smaller `resolution` lead to a more detailed map? Does it increase CPU usage?
* Experiment with other parameters and note the behaviour/changes of the quality of the mapping process. Which parameter changes improve mapping quality and which changes cause instability or worse maps.
