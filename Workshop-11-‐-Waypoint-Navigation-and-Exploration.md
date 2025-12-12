### Overview

In this workshop, you will:
 * use Nav2's waypoint follower node for waypoint (multi-goal) navigation
 * perform navigation using the [Simple Commander Python API](https://docs.nav2.org/commander_api/index.html) which allows you to send goals or waypoints programmatically. For an introductory, you can refer to this [video](https://www.youtube.com/watch?v=7pwfiM0pmlE).
 * explore an unknown environment using an exploration node with `Nav2` and `SLAM`

### Task 1 - Waypoint Navigation via Nav2 Rviz Plugin

* We can use the navigation stack for waypoint navigation, this can be done through the GUI of RViz or writing a custom node. 

* Let's start with extending [limo_navigation.launch.py](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/launch/limo_navigation.launch.py) that were added earlier in [Workshop 10](https://github.com/kivrakh/KV6022_limo_ros2/wiki/Workshop-10-%E2%80%90-Autonomous-Navigation) with integrating waypoint follower node. Open the launch file and add the following highlighted lines.
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
        'controller_server',
        'planner_server',
        'behaviour_server',
        'bt_navigator',
        <mark>'waypoint_follower',</mark>
    ]

    # LOAD PARAMETERS FROM YAML FILES
    param_bt_nav     = PathJoinSubstitution([limo_navigation, 'params', 'bt_nav.yaml'])
    param_planner    = PathJoinSubstitution([limo_navigation, 'params', 'planner.yaml'])
    param_controller = PathJoinSubstitution([limo_navigation, 'params', 'controller.yaml'])

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

    <mark>waypoint_follower = Node(
        package='nav2_waypoint_follower',
        executable='waypoint_follower',
        name='waypoint_follower',
        output='screen',
        parameters=[param_bt_nav],
    )</mark>

    # Planner Server Node
    node_planner = Node(
        package='nav2_planner',
        executable='planner_server',
        name='planner_server',
        output='screen',
        parameters=[param_planner],
        remappings=remappings,
    )

    # Controller Server Node
    node_controller = Node(
        package='nav2_controller',
        executable='controller_server',
        name='controller_server',
        output='screen',
        parameters=[param_controller],
        remappings=remappings,
    )

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
    ld.add_action(node_controller)
    <mark>ld.add_action(waypoint_follower)</mark>
    ld.add_action(node_lifecycle_manager2)

    return ld
</pre>

* Next, open the [bt_nav.yaml](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/params/bt_nav.yaml) file (used previously for `bt_navigator` and `behavior_server`) and append the `waypoint_follower` parameters.
<pre>
bt_navigator:
  ros__parameters:
    use_sim_time: True
    global_frame: map
    robot_base_frame: base_link
    odom_topic: odom
    bt_loop_duration: 10
    default_server_timeout: 20
    # 'default_nav_through_poses_bt_xml' and 'default_nav_to_pose_bt_xml' use defaults:
    # nav2_bt_navigator/navigate_to_pose_w_replanning_and_recovery.xml
    # nav2_bt_navigator/navigate_through_poses_w_replanning_and_recovery.xml
    # They can be set here or via a RewrittenYaml remap from a parent launch file to Nav2.
    default_bt_xml_filename: "nav2_bt_navigator/nav_to_pose_with_consistent_replanning_and_if_path_becomes_invalid.xml"

behavior_server:
  ros__parameters:
    costmap_topic: local_costmap/costmap_raw
    footprint_topic: local_costmap/published_footprint
    cycle_frequency: 10.0
    behavior_plugins: ["spin", "backup", "drive_on_heading", "assisted_teleop", "wait"]
    spin:
      plugin: "nav2_behaviors/Spin"
    backup:
      plugin: "nav2_behaviors/BackUp"
    drive_on_heading:
      plugin: "nav2_behaviors/DriveOnHeading"
    wait:
      plugin: "nav2_behaviors/Wait"
    assisted_teleop:
      plugin: "nav2_behaviors/AssistedTeleop"
    global_frame: odom
    robot_base_frame: base_link
    transform_tolerance: 0.1
    use_sim_time: true
    simulate_ahead_time: 2.0
    max_rotational_vel: 1.0
    min_rotational_vel: 0.4
    rotational_acc_lim: 3.2

<mark>waypoint_follower:
  ros__parameters:
    loop_rate: 20
    stop_on_failure: false
    action_server_name: "follow_waypoints"
    waypoint_task_executor_plugin: "wait_at_waypoint"
    wait_at_waypoint:
      plugin: "nav2_waypoint_follower::WaitAtWaypoint"
      enabled: True
      waypoint_pause_duration: 0</mark>
</pre>

* Perform `colcon build` and `source install/setup.bash` and check the launch file runs. We'll need four terminals, one for the simulation:
```
ros2 launch limo_gazebosim limo_gazebo_diff.launch.py
```
* One for running localisation and map_server nodes
```
ros2 launch limo_navigation limo_localisation.launch.py 
```
* Another another terminal for launching the `limo_navigation.launch.py` (planner + controller + BT nav + waypoint follower):
```
ros2 launch limo_navigation limo_navigation.launch.py
```
* In another terminal start Rviz
```
rviz2
```
* Make sure that the `Nav2 Goal` toolbar is added or visible at the top of RViz. If not, we can add it under the `+` sign. 
<p align="center">
<img width="660" height="220" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/nav2_goal.png" />
</p>

* Then switch Nav2 into Waypoint Following Mode via the `Navigation 2 panel`. This helpful Rviz tool that comes with `nav2` package (Panels->Add New Panel->nav2_rviz_plugins->Navigation2). 

<p align="center">
<img width="660" height="500" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/waypoint_mode.png" />
</p>

* To visualise waypoints, add a `MarkerArray` display in Rviz
   * Add → By topic → select `/waypoints` as a `MarkerArray` display

<p align="center">
<img width="400" height="600" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/waypoints_marker.png" />
</p>

* Using the `Nav2 Goal` or waypoint tool, click multiple locations on the map to define a list of waypoints: 

<p align="center">
<img width="600" height="600" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/waypoints_locations.png" />
</p>

* And when we are done with the waypoints we can start the waypoint navigation via the Nav2 panel in Rviz by clicking `Start Waypoint Following` button. The robot should drive through each waypoint in order:  

<p align="center">
<img width="700" height="500" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/waypoints_following.png" />
</p>

###  Task 2 - Custom Waypoints via Python (Simple Commander API)

Now you will use a Python script to send a sequence of waypoints programmatically.

* Inspect the example script [example_waypoint_follower.py](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/KV6022_assessment/KV6022_assessment/example_waypoint_follower.py). This script defines a list of waypoints and sends them to Nav2.

* **With simulation, localisation and navigation already running** (from Task 1), start the Python waypoint follower in a new terminal:
```
ros2 run KV6022_assessment example_waypoint_follower
```
The robot should autonomously move through the predefined waypoints.

* Now, you will define your own waypoint set utilising Task 1's Rviz waypoint generation to define waypoint locations. 
* Once you set the waypoints then inspect their coordinates via:
```
ros2 topic echo /waypoints --field pose.pose.position
```
This lets you see the `x`, `y` positions of each waypoint.

* Now adapt the Python [example_waypoint_follower.py](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/KV6022_assessment/KV6022_assessment/example_waypoint_follower.py) script with the updated list of `PoseStamped` waypoints.
 
* Another approach to collecting waypoints is
  * teleoperate the robot (`ros2 run teleop_twist_keyboard teleop_twist_keyboard`) to each location where you want a waypoint and run the following command while the robot is at that location to print the pose/coordinates of robot_pose of `base_link` in the `map` frame at that moment.
```
ros2 run tf2_ros tf2_echo map base_link
```
* Record `Translation` component of [`pos_x`, `pos_y`, `pos_z`] tuples for each pose printed in terminal, and add them to your route list in your script.
* Once you get the waypoint locations, you can run your modified script again and verify that the robot visits your chosen inspection points in sequence.

### Task 3: Exploration
Exploration is the process of autonomously moving in an unknown or partially known environment to gather new information. This is a real life use case for combining SLAM (mapping & localisation) with the navigation stack.

We will use [m-explore-ros2](https://github.com/robo-friends/m-explore-ros2.git) exploration package. Clone the package into your workspace with:
```
cd ~/KV6022_limo_ros2/src
git clone https://github.com/robo-friends/m-explore-ros2.git
```
Build the workspace and source the setup.bash to make sure ROS is aware about the new package:
```
cd ~/KV6022_limo_ros2
colcon build
source install/setup.bash
```

### Task 3.1 Configuration of explore node
As in the previous cases, download [explore_lite parameter configuration](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/params/explore.yaml) file and add it to the `param` folder of your package. This file contains the following parameters:
```yaml
/**:
  ros__parameters:
    robot_base_frame: base_link
    return_to_init: true
    costmap_topic: map
    costmap_updates_topic: map_updates
    visualize: true
    planner_frequency: 0.15
    progress_timeout: 30.0
    potential_scale: 3.0
    orientation_scale: 0.0
    gain_scale: 1.0
    transform_tolerance: 0.3
    min_frontier_size: 0.75
```
<details>
<summary><b>Explanation of the parameters</b></summary>
<ul>
<li><code>robot_base_frame</code> - the name of the base frame of the robot. This is used for determining robot position on map.</li>
<li><code>costmap_topic</code> - Specifies topic of source <code>nav_msgs/OccupancyGrid</code>.</li>
<li><code>costmap_updates_topic</code> - Specifies topic of source <code>map_msgs/OccupancyGridUpdate</code>. Not necessary if source of map is always publishing full updates, i.e. does not provide this topic.</li>
<li><code>visualize</code> - Specifies whether or not publish visualized frontiers.</li>
<li><code>planner_frequency</code> - Rate in Hz at which new frontiers will computed and goal reconsidered.</li>
<li><code>progress_timeout</code> - Time in seconds when robot do not make any progress for <code>progress_timeout</code>, current goal will be abandoned.</li>
<li><code>potential_scale</code> - Used for weighting frontiers. This multiplicative parameter affects frontier potential component of the frontier weight (distance to frontier).</li>
<li><code>orientation_scale</code> - Used for weighting frontiers. This multiplicative parameter affects frontier orientation component of the frontier weight.</li>
<li><code>gain_scale</code> - Used for weighting frontiers. This multiplicative parameter affects frontier gain component of the frontier weight (frontier size).</li>
<li><code>transform_tolerance</code> - Transform tolerance to use when transforming robot pose.</li>
<li><code>min_frontier_size</code> - Minimum size of the frontier to consider the frontier as the exploration goal. Value is in meter.</li>
</ul></div>
</details>

### Task 3.2 Launching exploration task
Next, let's include the an exploration node into `limo_navigation.launch.py` file that should look like this:

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
        'controller_server',
        'planner_server',
        'behaviour_server',
        'bt_navigator',
        'waypoint_follower',
    ]

    # LOAD PARAMETERS FROM YAML FILES
    param_bt_nav     = PathJoinSubstitution([limo_navigation, 'params', 'bt_nav.yaml'])
    param_planner    = PathJoinSubstitution([limo_navigation, 'params', 'planner.yaml'])
    param_controller = PathJoinSubstitution([limo_navigation, 'params', 'controller.yaml'])
    <mark>param_exploration= PathJoinSubstitution([limo_navigation, 'params', 'explore.yaml'])</mark>

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

    waypoint_follower = Node(
        package='nav2_waypoint_follower',
        executable='waypoint_follower',
        name='waypoint_follower',
        output='screen',
        parameters=[param_bt_nav],
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

    # Controller Server Node
    
    node_controller = Node(
        package='nav2_controller',
        executable='controller_server',
        name='controller_server',
        output='screen',
        parameters=[param_controller],
        remappings=remappings,
    )

    <mark>exploration_node = Node(
        package="explore_lite",
        name="explore_node",
        executable="explore",
        parameters=[param_exploration],
        output="screen",
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
    ld.add_action(node_controller)
    ld.add_action(waypoint_follower)
    <mark>ld.add_action(exploration_node)</mark>
    ld.add_action(node_lifecycle_manager2)

    return ld
</pre>

After build and source the workspace. Run simulation environment with world where the green areas not recognized as obstacle to see the results of exploration algorithm results more efficiently.
```
cd ~/KV6022_limo_ros2/src/limo_gazebosim/worlds
ros2 launch limo_gazebosim limo_gazebo_diff.launch.py world:=simulation_table.world
```
Then in another terminal run the slam launch (`limo_slam.launch.py`) from [Workshop 9](https://github.com/kivrakh/KV6022_limo_ros2/wiki/Workshop-9-%E2%80%90-Simultaneous-Localisation-and-Mapping-(SLAM)). If it is in `limo_localisation` package, use `limo_localisation` instead `limo_navigation` below:
```
ros2 launch limo_navigation limo_slam.launch.py
``` 
Then run launch navigation with exploration:
```
ros2 launch limo_navigation limo_navigation.launch.py 
```
* Finally run RViz for visualization `rviz2`. 
* If everything was set correctly exploration will start immediately after node initialization. Exploration will finish when whole area is discovered. An example result might look like this:

<p align="center">
<img width="512" height="250" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/exploration.gif" />
</p>

The exploration node will identify the boundaries of the know surrounding and will navigate the robot until it finds all the physical boundaries of the environment. 

* We can take a look on the `rqt_graph` of the simulation, but we'll see it's quite big! We can find the `explore_node` and we can see that it subscribes to the `/map` and to the `action status and feedback` of the navigation stack. 

<p align="center">
<img width="1200" height="800" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/rosgraph_exploration.png" />
</p>

### Summary

Congratulations on completing all the ROS 2 workshops!

I hope you enjoyed learning ROS 2 and learned a lot of practical skills. After finishing these workshops, controlling and navigating the robot should feel much confident and competent. Once you complete today's exercise, you'll be ready to begin your assessment.

Wishing you all the best on your assignments!
