### Overview
In this workshop, you will:
* Bring up the [Nav2](https://docs.nav2.org/) stack with your LIMO robot which will enables it to autonomously navigate to a goal pose. This process relies on the robot's ability to localize itself within an environment, sense its surroundings, plan an efficient path, and control its movements to follow that path successfully.
* Understand the roles of global planner (e.g. NavFn), local controller (e.g. DWB local planner), costmaps (global & local), configure RViz to visualise the navigation pipeline (map, laser, paths, costmaps, goal) and observing how parameter changes affect the behaviour of the planner and controller

### Task 1: Build a Navigation Package
  
We will utilise a new ROS2 package called `limo_navigation` for this week tasks, which will contain three additional directories `launch`, `params` and `maps`. Create the package and directories when you are in your workspace `src` folder `cd ~/KV6022_limo_ros2/src/`:

```
ros2 pkg create limo_navigation --build-type ament_python
cd limo_navigation
mkdir launch
mkdir params
mkdir maps
```
Your `setup.py` file should have some extra lines added to include the `launch` and `params` directories:
<pre>
from setuptools import find_packages, setup
<mark>import os</mark>
<mark>from glob import glob</mark>

package_name = 'limo_navigation'

setup(
    name=package_name,
    version='0.0.0',
    packages=find_packages(exclude=['test']),
    data_files=[
        ('share/ament_index/resource_index/packages',
            ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
        <mark>(os.path.join('share', package_name, 'launch'), glob(os.path.join('launch', '*launch.[pxy][yma]*'))),</mark>
        <mark>(os.path.join('share', package_name, 'params'), glob(os.path.join('params', '*.[yaml|txt|xml]*'))),
        (os.path.join('share', package_name, 'maps'), glob(os.path.join('maps', '*.[yaml|pgm|png]*'))),</mark>
        
    ],
    install_requires=['setuptools'],
    zip_safe=True,
    maintainer='mmmmm',
    maintainer_email='mmmm@mmm.com',
    description='TODO: Package description',
    license='TODO: License declaration',
    extras_require={
        'test': [
            'pytest',
        ],
    },
    entry_points={
        'console_scripts': [
        ],
    },
)
</pre>

A launch file will be built up gradually, whilst we add more param files to our navigation system. In the `launch` directory of the package, create a launch file called `limo_navigation.launch.py`. You can copy the code below to get you started.

```python
from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch.actions import IncludeLaunchDescription
from launch_ros.actions import SetParameter, Node
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch.substitutions import PathJoinSubstitution

def generate_launch_description():
    ld = LaunchDescription()

    # Parameters, Nodes and Launch files go here

    # Declare package directory
    limo_navigation = get_package_share_directory('limo_navigation')
    # Necessary fixes
    remappings = [('/tf', 'tf'), ('/tf_static', 'tf_static')]

    # LOAD PARAMETERS FROM YAML FILES
    param_bt_nav     = PathJoinSubstitution([limo_navigation, 'params', 'bt_nav.yaml'])

    # Behaviour Tree Navigator
    node_bt_nav = Node(
        package='nav2_bt_navigator',
        executable='bt_navigator',
        name='bt_navigator',
        output='screen',
        parameters=[param_bt_nav],
        remappings=remappings,
    )

    # Behaviour Tree Server
    node_behaviour = Node(
        package='nav2_behaviors',
        executable='behavior_server',
        name='behaviour_server',
        output='screen',
        parameters=[param_bt_nav],
        remappings=remappings,
    )

    # Add actions to LaunchDescription
    ld.add_action(SetParameter(name='use_sim_time', value=True))
    ld.add_action(node_bt_nav)
    ld.add_action(node_behaviour)

    return ld
```

Download [bt_nav.yaml](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/params/bt_nav.yaml) file (configuration for behaviour tree navigator) into the `params` directory. Then check everything builds as per usual.

```
colcon build
source install/setup.bash
```

### Task 2: Adding `Planner` (Path Planning) from `Nav2`
In the `Nav2` navigation stack, the so-called the `Planner` server consists of a ROS Node, which provides the general route from the start to the goal, whilst avoiding any known obstacles based on a map (path planning). The default planner plugin is `NavFn`. It contains path planning algorithms, either `Dijkstra` or `A*`. It is suitable for differential drive robots, therefore, suitable for our simulation. Each algorithm is suited to a particular design of robot, and may not support your configuration. You can check the full list of supported [Planners](https://navigation.ros.org/plugins/index.html#planners) in the documentation.

Along side robot pose estimates, the `Planner` requires a "global" costmap (i.e. it covers the entire global area you would wish to navigate in), this is handled by the planner server as well. The data to make the costmap and provide the start and end poses are of course all communicated via various ROS topics.

### Task 2.1: Writing the Planner Parameter File

The format of the configuration is taken from the [planner documentation](https://docs.nav2.org/configuration/packages/configuring-planner-server.html). An example `param` file for the `NavFn` package would look like the file below.

Create a parameter file named `planner.yaml` in the `params` directory. Copy the following example configuration into the `planner.yaml` file.

```yaml
planner_server:
  ros__parameters:
    expected_planner_frequency: 20.0
    use_sim_time: True
    planner_plugins: ["GridBased"]
    GridBased:
      plugin: "nav2_navfn_planner/NavfnPlanner"
      tolerance: 0.5
      use_astar: true
      allow_unknown: true
```

If you look under `plugin: "nav2\_navfn\_planner/NavfnPlanner"`, notice there are additional parameters `tolerance`, `use_astar`, `allow_unknown`. These parameters are specific to `NavFn` as per its [documentation](https://docs.nav2.org/configuration/packages/configuring-navfn.html). These options are explained in the table below.

| Option       | Default Value | Notes                                                         |
|-------------|---------------|---------------------------------------------------------------|
| tolerance   | 0.5           | Tolerance in meters between requested goal pose and end of path. |
| use_astar   | False         | Whether to use A*. If false, uses Dijkstra’s algorithm.       |
| allow_unknown | True        | Whether to allow planning in unknown space.                   |


### Task 2.2: Writing the Global Costmap Parameter File
For the global [costmap](https://docs.nav2.org/plugins/index.html#costmap-layers), we can simply use our map of the environment (`/map` topic) (as a static layer), an obstacle layer using the lidar (`/scan` topic) to catch objects before the static map updates, and include an inflation layer based on the robot size.

Open and edit the `planner.yaml` file where the `NavFn` parameters were added earlier and add the following highlighted lines.
<pre>
planner_server:
  ros__parameters:
    expected_planner_frequency: 20.0
    use_sim_time: True
    planner_plugins: ["GridBased"]
    GridBased:
      plugin: "nav2_navfn_planner/NavfnPlanner"
      tolerance: 0.5
      use_astar: true
      allow_unknown: true

<mark>global_costmap:
  global_costmap:
    ros__parameters:
      footprint_padding: 0.00
      update_frequency: 1.0
      publish_frequency: 1.0
      global_frame: map
      robot_base_frame: base_link
      use_sim_time: True
      robot_radius: 0.10 # radius set and used, so no footprint points
      resolution: 0.02
      plugins: ["static_layer", "obstacle_layer", "inflation_layer"]
      obstacle_layer:
        plugin: "nav2_costmap_2d::ObstacleLayer"
        enabled: True
        observation_sources: scan
        footprint_clearing_enabled: true
        max_obstacle_height: 2.0
        combination_method: 1
        scan:
          topic: /scan
          obstacle_max_range: 2.5
          obstacle_min_range: 0.0
          raytrace_max_range: 3.0
          raytrace_min_range: 0.0
          max_obstacle_height: 2.0
          min_obstacle_height: 0.0
          clearing: True
          marking: True
          data_type: "LaserScan"
          inf_is_valid: false
      static_layer:
        plugin: "nav2_costmap_2d::StaticLayer"
        map_subscribe_transient_local: True
        enabled: true
        subscribe_to_updates: true
        transform_tolerance: 0.1
      inflation_layer:
        plugin: "nav2_costmap_2d::InflationLayer"
        enabled: true
        inflation_radius: 0.10
        cost_scaling_factor: 1.0
        inflate_unknown: false
        inflate_around_unknown: true
      always_send_full_costmap: True </mark>
</pre>

There are various parameters associated with the costmap (e.g. `global_frame`, `use_sim_time`, `resolution`) but also for each layer there are additional parameters. It is clearly visible which parameters below to which section by the indentation scheme that these `xml` format files use. For a full list of costmap parameters check out the [costmap_2d github](https://github.com/ros-planning/navigation2/blob/3ed4c2dfa1ef9b31e117ccb5c35486b599e6b97e/nav2_costmap_2d/src/costmap_2d_ros.cpp#L90-L116).

The footprint of the robot is used to calculate if a robot can fit through gaps, and as part of the inflation of the costmap based on physical size of the robot. It is possible to declare a specific polygon for the footprint of the robot (e.g. four points could define a rectangular chassis), however, to keep things conceptually simpler we will only deal with a `robot_radius`.

### Task 2.3: Adding a Planner (Path Planning) to a Launch File

Open the `limo_navigation.launch.py` file and add the following highlighted lines.

<pre>
from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch.actions import IncludeLaunchDescription
from launch_ros.actions import SetParameter, Node
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch.substitutions import PathJoinSubstitution


def generate_launch_description():
    ld = LaunchDescription()

    # Parameters, Nodes and Launch files go here

    # Declare package directory
    limo_navigation = get_package_share_directory('limo_navigation')
    # Necessary fixes
    remappings = [('/tf', 'tf'), ('/tf_static', 'tf_static')]

    <mark>lifecycle_nodes = [
        'planner_server',
        'behaviour_server',
        'bt_navigator',
    ]</mark>

    # LOAD PARAMETERS FROM YAML FILES
    param_bt_nav     = PathJoinSubstitution([limo_navigation, 'params', 'bt_nav.yaml'])
    <mark>param_planner    = PathJoinSubstitution([limo_navigation, 'params', 'planner.yaml'])</mark>

    # Behaviour Tree Navigator
    node_bt_nav = Node(
        package='nav2_bt_navigator',
        executable='bt_navigator',
        name='bt_navigator',
        output='screen',
        parameters=[param_bt_nav],
        remappings=remappings,
    )

    # Behaviour Tree Server
    node_behaviour = Node(
        package='nav2_behaviors',
        executable='behavior_server',
        name='behaviour_server',
        output='screen',
        parameters=[param_bt_nav],
        remappings=remappings,
    )

    <mark># Planner Server Node
    node_planner = Node(
        package='nav2_planner',
        executable='planner_server',
        name='planner_server',
        output='screen',
        parameters=[param_planner],
        remappings=remappings,
    )</mark>

    <mark># Lifecycle Node Manager to automatically start lifecycles nodes (from list)
    node_lifecycle_manager2 = Node(
        package='nav2_lifecycle_manager',
        executable='lifecycle_manager',
        name='lifecycle_manager_navigation',
        output='screen',
        parameters=[{'autostart': True}, {'node_names': lifecycle_nodes}],
    )</mark>


    # Add actions to LaunchDescription
    ld.add_action(SetParameter(name='use_sim_time', value=True))
    ld.add_action(node_bt_nav)
    ld.add_action(node_behaviour)
    <mark>ld.add_action(node_planner)
    ld.add_action(node_lifecycle_manager2)</mark>

    return ld
</pre>

### Task 2.4: Map and Localisation

* Download the [map.yaml](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/maps/map.yaml) and [map.pgm](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/maps/map.pgm) and place them under your package's `maps` folder
* Download [limo_localisation.launch.py](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/launch/limo_localisation.launch.py) into `limo_navigation/launch/` and update `pkg_name` inside to `limo_navigation`.

* Perform the usual `colcon build` and `source install/setup.bash`.

Run Gazebo simulation environment and check the launch file runs:
```
ros2 launch limo_gazebosim limo_gazebo_diff.launch.py
```
In a new terminal, run localisation and map_server nodes
```
ros2 launch limo_navigation limo_localisation.launch.py 
```
In a new terminal, run Rviz visualisation
```
rviz2
```
AMCL needs an initial guess so use `2D Pose Estimate` button on Rviz to set approximately where the robot is.
In a new terminal, run
```
ros2 launch limo_navigation limo_navigation.launch.py 
```
If everything is running correctly, in `rviz` it should be possible to view the global costmap topic (`/global_costmap/costmap`) similar to the image below. Note that the specific colour palette comes from selecting `costmap` as the `Color Scheme`. Notice how obstacles got inflated by a safety area which is not traversable by the robot. You can read more about it [here](http://wiki.ros.org/costmap_2d).

<p align="center">
<img width="771" height="536" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/rviz_costmap.png" />
</p>

> [!WARNING]
> **DDS / RMW choice for Nav2**
>
> By default, ROS 2 on Humble typically uses **Fast DDS** as the middleware
> (RMW implementation). In many cases this works fine, but it is known that
> Fast DDS can sometimes cause issues with **Nav2** 
>
> If you see missing /map, /tf, costmap topics, or Nav2 nodes not reacting, consider switching to Cyclone DDS `rmw_cyclonedds_cpp` by:
> `echo export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp >> ~/.bashrc`

### Task 3: Adding `Controller` (Path Following) from Nav2

The `Controller` plugin generates velocity commands to move the robot along the planned path. We have list of "plugins" to choose from, allowing us flexibility over what controller architecture to use. In fact, you can write your [own controller](https://docs.nav2.org/plugin_tutorials/index.html) and planner! For a full list of \texttt{Controllers} available, visit the `Nav 2 Controller` [documentation](https://docs.nav2.org/plugins/index.html#controllers). 

For this workshop, we will use the `Dynamic Window Approach` (`DWB` planner) controller.

These `Controllers` can be tuned or altered to provide the behaviour you desire (e.g. how faithfully should it stick to the global path). It is not possible to exhaustively cover this tuning for every controller here, but most provide a tuning guide of sorts. For generic advice, please take a look at [Nav 2 Tuning Guide](https://docs.nav2.org/tuning/index.html)

### Task 3.1: Writing the Controller Parameter File}
The format of the configuration is taken from the [controller documentation](https://docs.nav2.org/configuration/packages/configuring-controller-server.html#configuring-controller-server). It consists of general controller parameters, but controller plugin specific parameters can also be passed. For `DWB` local planner some of these parameters can be [found here](https://docs.nav2.org/configuration/packages/dwb-params/controller.html).

Create a file named `controller.yaml` in the `params` directory. Include the example configuration below:

```yaml
controller_server:
  ros__parameters:
    use_sim_time: True
    controller_frequency: 20.0
    min_x_velocity_threshold: 0.001
    min_y_velocity_threshold: 0.5
    min_theta_velocity_threshold: 0.001
    failure_tolerance: 0.3
    odom_topic: "odom"
    progress_checker_plugins: ["progress_checker"] 
    goal_checker_plugin: "goal_checker"
    controller_plugins: ["FollowPath"] # This is where we define the DWB controller plugin
    progress_checker:
      plugin: "nav2_controller::SimpleProgressChecker"
      required_movement_radius: 0.5
      movement_time_allowance: 10.0
    goal_checker:
      plugin: "nav2_controller::SimpleGoalChecker"
      xy_goal_tolerance: 0.25
      yaw_goal_tolerance: 0.25
      stateful: True
    FollowPath:
      plugin: "dwb_core::DWBLocalPlanner"
      debug_trajectory_details: True
      min_vel_x: 0.0
      min_vel_y: 0.0
      max_vel_x: 0.26
      max_vel_y: 0.0
      max_vel_theta: 1.0
      min_speed_xy: 0.0
      max_speed_xy: 0.26
      min_speed_theta: 0.0
      acc_lim_x: 2.5
      acc_lim_y: 0.0
      acc_lim_theta: 3.2
      decel_lim_x: -2.5
      decel_lim_y: 0.0
      decel_lim_theta: -3.2
      vx_samples: 20
      vy_samples: 5
      vtheta_samples: 20
      sim_time: 1.7
      linear_granularity: 0.05
      angular_granularity: 0.025
      transform_tolerance: 0.2
      xy_goal_tolerance: 0.25
      trans_stopped_velocity: 0.25
      short_circuit_trajectory_evaluation: True
      stateful: True
      critics: ["RotateToGoal", "Oscillation", "BaseObstacle", "GoalAlign", "PathAlign", "PathDist", "GoalDist"]
      BaseObstacle.scale: 0.02
      PathAlign.scale: 32.0
      PathAlign.forward_point_distance: 0.1
      GoalAlign.scale: 24.0
      GoalAlign.forward_point_distance: 0.1
      PathDist.scale: 32.0
      GoalDist.scale: 24.0
      RotateToGoal.scale: 32.0
      RotateToGoal.slowing_factor: 5.0
      RotateToGoal.lookahead_time: -1.0
```
The `DWB` planner has many options to tune the system. It is recommended that for your own robot, you would carefully read about all the different options and see which need to be altered. 

Notice as well we needed to include a `goal_checker` and a `progress_checker`, these do pretty much what you would expect. By having these separate (rather than having them included in the `controller` plugin), it again allows for modularity. These can be tuned as well, for example, in the goal checker we set the tolerance of how close the robot must be to the goal (as it is nearly impossible for a robot to drive exactly to the point asked of it).

### Task 3.2: Writing the Local Costmap Parameter File
The local costmap in comparison to the global costmap, only covers a small portion around the robot (which is quicker to compute), and is updated more regularly. In this case, we do not necessarily need the static map, just obstacles and other threats we wish to track. Because we are using layered costmaps, it is possible to add obstacle (or other types) of layers representing different sensors. We'll just add the lidar for obstacles this time. Again we include the inflation layer to convert obstacles into the configuration space of the robot.

Open the `controller.yaml` file where the `DWB` parameters were added earlier. Append the following local costmap configuration to the file.

<pre>
controller_server:
  ros__parameters:
    # controller server parameters (see Controller Server for more info)
    use_sim_time: True
    controller_frequency: 20.0
    min_x_velocity_threshold: 0.001
    min_y_velocity_threshold: 0.5
    min_theta_velocity_threshold: 0.001
    progress_checker_plugin: "progress_checker"
    goal_checker_plugins: ["goal_checker"]
    controller_plugins: ["FollowPath"]
    progress_checker:
      plugin: "nav2_controller::SimpleProgressChecker"
      required_movement_radius: 0.5
      movement_time_allowance: 10.0
    goal_checker:
      plugin: "nav2_controller::SimpleGoalChecker"
      xy_goal_tolerance: 0.25
      yaw_goal_tolerance: 0.25
      stateful: True
    # DWB controller parameters
    FollowPath:
      plugin: "dwb_core::DWBLocalPlanner"
      debug_trajectory_details: True
      min_vel_x: 0.0
      min_vel_y: 0.0
      max_vel_x: 0.1
      max_vel_y: 0.0
      max_vel_theta: 0.5
      min_speed_xy: 0.0
      max_speed_xy: 0.26
      min_speed_theta: 0.0
      acc_lim_x: 0.5
      acc_lim_y: 0.0
      acc_lim_theta: 0.5
      decel_lim_x: -0.5
      decel_lim_y: 0.0
      decel_lim_theta: -0.5
      vx_samples: 20
      vy_samples: 5
      vtheta_samples: 20
      sim_time: 3.0
      linear_granularity: 0.05
      angular_granularity: 0.025
      transform_tolerance: 0.2
      xy_goal_tolerance: 0.25
      trans_stopped_velocity: 0.25
      short_circuit_trajectory_evaluation: True
      stateful: True
      critics: ["RotateToGoal", "Oscillation", "BaseObstacle", "GoalAlign", "PathAlign", "PathDist", "GoalDist"]
      BaseObstacle.scale: 0.02
      PathAlign.scale: 32.0
      GoalAlign.scale: 24.0
      PathAlign.forward_point_distance: 0.1
      GoalAlign.forward_point_distance: 0.1
      PathDist.scale: 32.0
      GoalDist.scale: 24.0
      RotateToGoal.scale: 32.0
      RotateToGoal.slowing_factor: 5.0
      RotateToGoal.lookahead_time: -1.0

<mark>local_costmap:
  local_costmap:
    ros__parameters:
      update_frequency: 5.0
      publish_frequency: 2.0
      global_frame: odom
      robot_base_frame: base_link
      use_sim_time: True
      rolling_window: True
      width: 2
      height: 2
      resolution: 0.05
      robot_radius: 0.175
      plugins: ["obstacle_layer", "inflation_layer"]
      always_send_full_costmap: True
      inflation_layer:
        plugin: "nav2_costmap_2d::InflationLayer"
        cost_scaling_factor: 4.0
        inflation_radius: 0.45
      obstacle_layer:
        plugin: "nav2_costmap_2d::ObstacleLayer"
        enabled: True
        observation_sources: scan_source
        scan_source:
          topic: /scan
          max_obstacle_height: 2.0
          clearing: True
          marking: True
          data_type: "LaserScan"
          raytrace_max_range: 3.0
          raytrace_min_range: 0.0
          obstacle_max_range: 2.5
          obstacle_min_range: 0.0</mark>
</pre>

### Task 3.3: Adding a Controller to the Launch File
Finally, we need to add the controller node to the launch file, along with including the config file and ensuring the lifecycle manager knows to handle it. 

Open the `limo_navigation.launch.py` file and add the highlighted lines.

<pre>
from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch.actions import IncludeLaunchDescription
from launch_ros.actions import SetParameter, Node
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch.substitutions import PathJoinSubstitution

def generate_launch_description():
    ld = LaunchDescription()

    # Parameters, Nodes and Launch files go here

    # Declare package directory
    limo_navigation = get_package_share_directory('limo_navigation')
    # Necessary fixes
    remappings = [('/tf', 'tf'), ('/tf_static', 'tf_static')]

    lifecycle_nodes = [
        <mark>'controller_server',</mark>
        'planner_server',
        'behaviour_server',
        'bt_navigator',
    ]

    # LOAD PARAMETERS FROM YAML FILES
    param_bt_nav     = PathJoinSubstitution([limo_navigation, 'params', 'bt_nav.yaml'])
    param_planner    = PathJoinSubstitution([limo_navigation, 'params', 'planner.yaml'])
    <mark>param_controller = PathJoinSubstitution([limo_navigation, 'params', 'controller.yaml'])</mark>

    # Behaviour Tree Navigator
    node_bt_nav = Node(
        package='nav2_bt_navigator',
        executable='bt_navigator',
        name='bt_navigator',
        output='screen',
        parameters=[param_bt_nav],
        remappings=remappings,
    )

    # Behaviour Tree Server
    node_behaviour = Node(
        package='nav2_behaviors',
        executable='behavior_server',
        name='behaviour_server',
        output='screen',
        parameters=[param_bt_nav],
        remappings=remappings,
    )

     # Planner Server Node
    node_planner = Node(
        package='nav2_planner',
        executable='planner_server',
        name='planner_server',
        output='screen',
        parameters=[param_planner],
        remappings=remappings,
    )

    <mark># Controller Server Node
    node_controller = Node(
        package='nav2_controller',
        executable='controller_server',
        name='controller_server',
        output='screen',
        parameters=[param_controller],
        remappings=remappings,
    )</mark>

    # Lifecycle Node Manager to automatically start lifecycles nodes (from list)
    node_lifecycle_manager2 = Node(
        package='nav2_lifecycle_manager',
        executable='lifecycle_manager',
        name='lifecycle_manager_navigation',
        output='screen',
        parameters=[{'autostart': True}, {'node_names': lifecycle_nodes}],
    )

    # Add actions to LaunchDescription
    ld.add_action(SetParameter(name='use_sim_time', value=True))
    ld.add_action(node_bt_nav)
    ld.add_action(node_behaviour)
    ld.add_action(node_planner)
    <mark>ld.add_action(node_controller)</mark>
    ld.add_action(node_lifecycle_manager2)

    return ld
</pre>

* Perform `colcon build` and `source install/setup.bash` and check the launch file runs.

### Task 4: Sending Goals to the Robot Navigation Stack (Navigation on a known map)

* Now everything is ready, start the simulation, 
```
ros2 launch limo_gazebosim limo_gazebo_diff.launch.py
```

* Start AMCL localization in the given map
```
ros2 launch limo_localisation limo_localisation.launch.py
```

* And the navigation stack:

```
ros2 launch limo_navigation limo_navigation.launch.py 
```

Open a new terminal and perform the command for Navigation Action Servers
```
ros2 action list
```

It should return something like:
<pre>
/backup
/compute_path_through_poses
/compute_path_to_pose
/drive_on_heading
/follow_path
/navigate_through_poses
/navigate_to_pose
/spin
/wait
</pre>

These `Action Servers` are all supplied by the various plugins.

There is interdependancy between these action servers also, for example, the user would make a call to `/navigation_to_pose`, which then uses `compute_path_to_pose` (Planner) followed by `follow_path` (Controller). It can be a bit of a tangled mess at first glance, but in reality its more linear than it appears.

In a new terminal, run the command 

```
ros2 run rqt_graph rqt_graph
```

In the top left corner, select `Nodes/Topics (all)` to get a total overview of all the connections between nodes and their topics.

Press the `refresh` arrows button to update the window. Hover over the oval containing `/planner_server`, the green arrows (published topics) go to `/compute_path_to_pose/_action`, `/compute_path_through_pose/_action` and a topic called `/plan`.

The `/controller_server` node is similar but provides the `/follow_path/_action` topics, a `/local_plan` and most importantly the `/cmd_vel` velocity commands to the robot.

<p align="center">
<img width="1173" height="650" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/rosgraph_nav2.png" />
</p>

### Task 4.1: Setup RViz Configuration

To visualize the `Nav2` stack in RVIZ, set `Global Frame` to map and include followings:

- The robot’s current position and orientation  
  - Either using `RobotModel` or `Axes` display in RViz

- LaserScan Topic: `/scan`

- Map Topic: `/map`

- Costmap Topics:
  - `/global_costmap/costmap` – the global costmap (obstacles and inflation zones)
  - `/local_costmap/costmap` – the local costmap

- Path Topics:
  - `/plan` – the planned global path
  - `/local_plan` – the local trajectory being executed
  - Change colours so you can differentiate between the paths.

### Task 4.2: Send a Goal Using Visual Tools

To send the robot to a goal we need to provide a goal to the `/navigation_to_pose` action server. An autonomous algorithm/agent would publish goal messages directly to the action server. We would select a point on the map in Rviz, along the top bar there is a button called `2D Goal Pose` or `Nav 2 Goal`. Try this out with the steps below.

* Press the `2D Goal Pose` or `Nav 2 Goal` button to enable the tool
* Hover over a specific point in the map you wish to navigate to
* Press and HOLD the left mouse button
* Drag your mouse around to change the direction of the arrows
* Release the left mouse button

The base of the arrow indicates the pose position, whereas the arrow indicates the pose orientation. Once you release the left mouse button, the goal is sent.

The robot should be driving to where your arrow was, whilst publishing the global path and the trajectory the controller is attempting to take.

<p align="center">
<img width="512" height="210" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/nav2_demo.gif" />
</p>

### Task 4.3: Send a Goal Pose Manually
To send the robot to a goal we need to provide a goal to the `/navigation_to_pose` action server. An autonomous algorithm/agent would publish goal messages directly to the action server. We will send messages in the terminal to emulate this.

Ensure you have RVIZ visible on the screen, and in a new terminal (placed somewhere as to not block your view of RVIZ) send the command below.

```
ros2 action send_goal /navigate_to_pose nav2_msgs/action/NavigateToPose "pose:
  header:
    stamp:
      sec: 0
      nanosec: 0
    frame_id: ''
  pose:
    position:
      x: 1.0
      y: 0.0
      z: 0.0
    orientation:
      x: 0.0
      y: 0.0
      z: 0.0
      w: 1.0
behavior_tree: ''"
```

The robot should be navigating! You should see the robot drive to `x: 1.0` and `y: 0.0` location, whilst publishing the global path and the trajectory the controller is attempting to take.


###  Task 5 - Obstacle avoidance and Parameter Change

* Navigate close to the map borders and investigate if the robot can operate safely in the presence of borders.
* Place an obstacle in front of the robot, and in Rviz issue a navigation goal in a straight line behind the obstacle. 
* The parameter values can be dynamically reconfigured on the fly by opening `ros2 run rqt_reconfigure rqt_reconfigure` Please select `global_costamp` or `local_costamp` in the menu on the left, and you should be able to visualise all the parameters you can change.
* Modify the `global_costmap`'s `inflation_radius` parameter to slightly larger or smaller values.
* Modify `Controller` parameters for a cautious velocity limits with `max_vel_x: 0.15` and `max_vel_theta: 0.6` or to aggressive profile with `max_vel_x: 0.35` and `max_vel_theta: 1.3`.
* Similarly modify `Planner` params of `tolerance: 0.5` and `allow_unknown: false` or `tolerance: 0.1` and `allow_unknown: true`
* Note the differences in the planning and controller behaviour. Inspect the optimisation potential in Rviz.