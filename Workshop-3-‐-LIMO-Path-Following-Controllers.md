### Scope of the Workshop
In this workshop, you will implement and test three different control algorithms for autonomous path following:
1. On/Off Controller
2. Proportional (P) Controller
3. Kinematic Controller

You'll implement them in a single ROS 2 package containing a Python node that:
* Subscribes to `/waypoint_cmd`: receives the next waypoint that the robot needs to travel (published by the `trajectory_publisher` node).
* Subscribes to `/odom`: gets the robot's current position and orientation according to the robot's odometry.
* Publishes to `/cmd_vel`: sends velocity commands to drive the robot to each point in the path.

### 1. Environment Setup
* Pull changes from the [repo](https://github.com/kivrakh/KV6022_limo_ros2) and instal dependencies:
```
git pull origin main
sudo apt install ros-humble-tf-transformations
```
* Build and Source the workspace
> [!TIP]
> Source the setup file to allow commands like `ros2 run` work with it. You can add the source command to your shell startup script`~/.bashrc` file so you don’t need to retype or have to source the setup file every time you open a new shell:
```
echo "source ~/KV6022_limo_ros2/install/setup.bash" >> ~/.bashrc
```
### 2. Launch the Simulator and Trajectory publisher
* Run Gazebo first, then in a new terminal you are going to start the `trajectory_publisher` node.
* The trajectory publisher publishes the target waypoints to the `/waypoint_cmd` topic.
* There are two modes (`dis` and `dor`) and three routes (`route1`, `route2`, and `route3`): use ONE OF the following. If you use `dor`, the referee expects you to achieve both target orientation and position, whereas `dis` only expects you to achieve the target position. Make sure your robot can solve the task with `dis` before you try `dor`. `route3` can be considered challenging for those who like to be challenged.
```
ros2 run trajectory_referee trajectory_publisher route1 dis
```
```
ros2 run trajectory_referee trajectory_publisher route1 dor
```

The gazebo and trajectory_publisher will start publishing and subscribing the following topics. Check topic details with: `ros2 topic echo`, `ros2 topic info` and `ros2 interface show` commands.
> [!NOTE]  
> Note the following topic names for your publishers and subscribers:
>|Data | Topic | Message Type | Notes|
>-- | -- | -- | --
>|Next waypoint | `/waypoint_cmd` | `geometry_msgs/Transform` | Provided by trajectory_publisher|
>|Robot odometry | `/odom` | `nav_msgs/Odometry` | Pose + twist|
>|Velocity command | `/cmd_vel` | `geometry_msgs/Twist` | Publish here|
>|Route visualization | `/visualization_marker_array` | visualization_msgs/MarkerArray | For RViz display|

### 3. Visualisation and Exploration
* You will be required both the simulator and the trajectory_publisher node running while you test your code. In order to see what your robot sees, in a new terminal window run
```
ros2 launch trajectory_skeleton view_robot.launch.py
```
<img src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/rviz_waypoints.png" width="800">

The visualisation will show the output of the robot's self-model relative to its odometry estimation of its original location (`/odom`) as maintained by Gazebo simulaton node and the trajectory waypoint markers (a colourful path of small arrows).

In particular, rviz 2 should be subscribed to a new topic provided by the referee called `/visualization_marker_array` as Marker Array). 
> [!NOTE]  
> To drive the robot around in its simulated world, in a separate terminal:
> ```
> ros2 run teleop_twist_keyboard teleop_twist_keyboard
> ```
> It is recommended that you do this in order to investigate the problem that you will need to solve with your
controller.

### 4. Run the Skeleton Controller

Once you are familiar with the visualisation and the robot teleoperation, you can run the skeleton code:
```
ros2 run trajectory_skeleton trajectory_tracking_controller
```
The best way to investigate this setup is to have different terminals/windows arrayed across your workspace:
* Gazebo simulator (to see what the robot is really doing).
* The RViz visulaisation (to see what the robot sees).
* The output of the trajectory_publisher node (to report on your progress).
* The output of your controller (for debugging it).
* The keyboard teleoperator (for you to guide the robot yourself).
> [!TIP]
> Once the referee does start keeping track, if you are using rviz you will see the reached markers dim, and the referee will also print to the terminal as each way-point is reached.

### 5. Implement Your Controllers
Now you are in a position to edit the controller file `trajectory_tracking_controller.py`, and run the controller (as above) to see how it performs. Your code would normally go at the bottom of the file `trajectory_tracking_controller.py` where it says `DRIVE THE
ROBOT HERE`. You will need to make use of the variables calculated above that point in the code called `waypoint.translation.x`, `waypoint.translation.y`, `waypoint_theta`, `robot_pose.getOrigin().x`, `robot_pose.getOrigin().y`, and `robot_theta`.

1. Controller 1 – **On/Off Controller**
* Implement the on-off algorithm:
 * If the position or heading error is larger than a small threshold → drive the robot.
 * If the error is within the threshold → stop the robot.
* Implement position and heading thresholds.
* Tune the threshold range and observe robot behaviour on `route1`, `route2`, `route3` using `dis` mode

2. Controller 2 – **Proportional (P) Controller**
* Now move to a Proportional control approach where control signals are proportional to the error magnitude.
* Derive control commands:
   * $v = K_p \cdot e_{distance}$ , $\omega = K_p \cdot e_{angle}$
* Experiment with different K_p gains for smooth yet responsive motion.
* Compare performance against the On/Off controller.
* Repeat the experiment with `route1`, `route2`, `route3` using `dis`

3. Controller 3 – **Kinematic Position Controller**
* Finally, implement a kinematic controller based on the differential-drive model:
   * $v = K_{\rho} \cdot \rho$ , $\omega = K_{\alpha} \cdot \alpha + K_{\beta} \cdot \beta$
  where
   * $\rho$: distance to goal
   * $\alpha$: heading to goal relative to robot
   * $\beta$: goal orientation error
* Compute $\rho$, $\alpha$, $\beta$ using odometry and waypoint transform.
* Experiment with different $K_{\rho}$, $K_{\alpha}$, $K_{\beta}$ gains for smooth approaches the reference distance and orientation.
* Repeat the experiment with `route1`, `route2`, `route3` using `dor` mode