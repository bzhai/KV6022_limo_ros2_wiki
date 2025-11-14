### Overview
In this lab, you will gain practical experience using an Extended Kalman Filter (EKF) for robot localization (e.g., position tracking, navigation by odometry) with the `robot_localization` package. By completing this workshop, you will experiment with multi-sensor fusing of noisy wheel encoder odometry data, laser and IMU sensor data. By doing so, observing how sensor fusion can improve position tracking accuracy in scenarios involving wheel slip, turns, obstacle collision and other disturbances.

### Preparation 1: Setting wheel encoder sensor
You need modify the LIMO robot gazebo configuration to use wheel encoder odometry as the source for odometry information to reflect the motion under noise.
* Open the LIMO robot gazebo configuration file [limo_diff.gazebo](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/limo_description/urdf/limo_diff/limo_diff.gazebo). In the `libgazebo_ros_diff_drive.so` plugin set the `odometry_source` parameter to `0` to use odometry source using the wheel encoder sensor: 
<pre>
&lt;gazebo&gt;
    &lt;plugin name="four_diff_controller" filename=<mark>"libgazebo_ros_diff_drive.so"</mark>&gt;
        &lt;update_rate&gt;30&lt;/update_rate&gt;
        &lt;num_wheel_pairs&gt;2&lt;/num_wheel_pairs&gt;
        &lt;left_joint&gt;front_left_wheel&lt;/left_joint&gt;
        &lt;right_joint&gt;front_right_wheel&lt;/right_joint&gt;
        &lt;left_joint&gt;rear_left_wheel&lt;/left_joint&gt;
        &lt;right_joint&gt;rear_right_wheel&lt;/right_joint&gt;
        &lt;wheel_separation&gt;0.172&lt;/wheel_separation&gt;
        &lt;wheel_diameter&gt;0.09&lt;/wheel_diameter&gt;
        &lt;max_wheel_torque&gt;20&lt;/max_wheel_torque&gt;
        &lt;max_wheel_acceleration&gt;1.0&lt;/max_wheel_acceleration&gt;
        &lt;command_topic&gt;cmd_vel&lt;/command_topic&gt;
        &lt;publish_odom&gt;true&lt;/publish_odom&gt;
        <mark>&lt;publish_odom_tf&gt;false&lt;/publish_odom_tf&gt;</mark>
        &lt;publish_wheel_tf&gt;false&lt;/publish_wheel_tf&gt;
        &lt;odometry_topic&gt;odom&lt;/odometry_topic&gt;
        &lt;odometry_frame&gt;odom&lt;/odometry_frame&gt;
        &lt;robot_base_frame&gt;base_footprint&lt;/robot_base_frame&gt;
        &lt;!-- Odometry source, 0 for ENCODER, 1 for WORLD, defaults to WORLD --&gt;
        <mark>&lt;odometry_source&gt;0&lt;/odometry_source&gt;</mark>
        <mark>&lt;ros&gt;</mark>
            <mark>&lt;remapping&gt;odom:=/odom/wheel&lt;/remapping&gt;</mark>
        <mark>&lt;/ros&gt;</mark>
    &lt;/plugin&gt;
&lt;/gazebo&gt;
</pre>
* Also, copy paste the following contents of `libgazebo_ros_p3d.so` plugin just after previous plugin to get the robot's actual (ground truth) position to compare the EKF estimation. This way,the ground truth position will be published to `odom/perfect` topic.
<pre>
&lt;gazebo&gt;
    &lt;plugin name="limo_diff_drive_perfect" filename=<mark>"libgazebo_ros_p3d.so"</mark>&gt;
        &lt;always_on&gt;true&lt;/always_on&gt;
        &lt;update_rate&gt;30.0&lt;/update_rate&gt;
        &lt;body_name&gt;base_link&lt;/body_name&gt;
        &lt;topic_name&gt;odom/perfect&lt;/topic_name&gt;
        &lt;gaussian_noise&gt;0.01&lt;/gaussian_noise&gt;
        <mark>&lt;frame_name&gt;odom&lt;/frame_name&gt;</mark>
        &lt;xyz_offset&gt;0 0 0&lt;/xyz_offset&gt;
        &lt;rpy_offset&gt;0 0 0&lt;/rpy_offset&gt;
        &lt;ros&gt;
           <mark>&lt;remapping&gt;odom:=/odom/perfect&lt;/remapping&gt;</mark>
        &lt;/ros&gt;</mark>
    &lt;/plugin&gt;
&lt;/gazebo&gt;
</pre>
* To set the robot's initial pose (position and orientation) to zero, update the following parameters in [limo_gazebo_diff.launch.py](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/limo_gazebosim/launch/limo_gazebo_diff.launch.py) 
<pre>
spawn_x_val = <mark>'0.0'</mark>
spawn_y_val = <mark>'0.0'</mark>
spawn_z_val = <mark>'0.0'</mark>
spawn_yaw_val = <mark>'0.0'</mark>
</pre>

* Build and source your workspace
* Launch the Gazebo simulation
* Verify the wheel encoder odometry is in  place. Move robot in forward towards to the wall, that is we expect increasing its `x` value. Observe that even robot is not moving due to the collosion to the wall, `x` position of the robot keep increasing due to fact that robot is considering it is moving forward according wheels turns.

```
ros2 topic echo /odom/wheel --field pose.pose.position
```
<p align="center">
<img width="800" height="400" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/wheel_odom_verify.png" />
</p>

### Preparation 2: Adding noise to sensor measurements (wheel odometry, imu)
* You will add noise to the odometry data to better reflect real-world scenario as Gazebo provides near-perfect sensory model. Review [noisy_odometry.py](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/example_codes/noisy_odometry.py) node which will add gaussion noise to odometry motion model similar to reflect below uncertainty in motion model explained [here](https://blog.lxsang.me/post/id/16).
 
<p align="center">
<img width="620" height="440" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/odometry_motion%20model.png" />
</p>

* Inspect IMU sensor model in the LIMO's [gazebo model](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/limo_description/urdf/limo_gazebo.xacro). Search for the `<sensor name="imu_sensor" type="imu">` tag where you will see the `<noise>` tag under the `<orientation>`, `<angular_velocity>` and `<linear_acceleration>` tags. IMU noise is already configured.

### Preparation 3: Laser-based Odometry Package
The [laser_scan_matcher](https://wiki.ros.org/laser_scan_matcher) package is operating scan match between consecutive laser scan messages to calculate estimated position of the laser. Thus, it will serve as a laser-based odometry estimator besides the wheel encoder odometry estimation. 
* To use the package, clone the `ros2_laser_scan_matcher` and its dependent `csm` repository into your workspace:
```
cd ~/KV6022_limo_ros2/src
git clone https://github.com/AlexKaravaev/ros2_laser_scan_matcher.git
git clone https://github.com/AlexKaravaev/csm.git
```
* Build and source the workspace
* Run the node to publish `/odom/laser` topic via:
```
ros2 run ros2_laser_scan_matcher laser_scan_matcher --ros-args -p publish_odom:=/odom/laser -p publish_tf:=false -p laser_frame:=laser_link
```
* Verify the laser odometry is being published `ros2 topic echo /odom/laser --field pose.pose.position`

### Task1: Setting up Sensor Fusion Package (`robot_localization`)
We have odometry estimation coming from wheel encoders, imu and laser sensor. These sources will be combined through EKF algorithm to achieve better odometry and hence better position tracking. `robot_localization` [package](http://docs.ros.org/en/noetic/api/robot_localization/html/index.html) is the implementation of EKF which you will configure it to fuse noisy wheel odometry, IMU data and laser odometry for obtaining combined odometry.
* First of all, clone the `robot_localization` package repository (humble branch) into your `KV6022_limo_ros2` workspace:
```
cd ~/KV6022_limo_ros2/src
git clone -b humble-devel https://github.com/cra-ros-pkg/robot_localization.git
```
* Install dependency of a particular package with,
```
cd ..
sudo rosdep init
rosdep update
rosdep install --from-paths src --ignore-src -r -y
```
* Build only `robot_localization` package via `--packages-select` argument and source the package: 
```
 colcon build --symlink-install --packages-select robot_localization
 source install/setup.bash 
```
* Configure the EKF node by creating and saving the following configuration file as a `ekf_limo.yaml` file to under the `robot_localization` package `params` directory. If you want to learn more about the explanations of each parameter, refer to `ekf.yaml` in the same directory or its [documentation](http://docs.ros.org/en/melodic/api/robot_localization/html/state_estimation_nodes.html#predict-to-current-time):
<pre>
ekf_filter_node:
    ros__parameters:
        frequency: 30.0
        two_d_mode: true

        odom_frame: odom
        base_link_frame: base_link
        world_frame: odom
        predict_to_current_time: true

        initial_state: [<mark>0.0</mark>,  <mark>0.0</mark>,  0.0,
                        0.0,  0.0,  <mark>0.0</mark>,
                        0.0,  0.0,  0.0,
                        0.0,  0.0,  0.0,
                        0.0,  0.0,  0.0]

        odom0: /odom/wheel/noisy
        odom0_config: [<mark>true</mark>, <mark>true</mark>, false,
                      false, false, <mark>true</mark>,
                      false,  false,  false,
                      false, false, <mark>true</mark>,
                      false, false, false]
        #             [x_pos   , y_pos    , z_pos,
        #             roll    , pitch    , yaw,
        #             x_vel   , y_vel    , z_vel,
        #             roll_vel, pitch_vel, yaw_vel,
        #             x_accel , y_accel  , z_accel]
        
        odom0_differential: true
        odom0_queue_size: 10

        odom1: /odom/laser
        odom1_config: [<mark>true</mark>, <mark>true</mark>, false,
                      false, false, <mark>true</mark>,
                      false,  false,  false,
                      false, false, <mark>true</mark>,
                      <mark>true</mark>, <mark>true</mark>, false]
        #        [x_pos   , y_pos    , z_pos,
        #         roll    , pitch    , yaw,
        #         x_vel   , y_vel    , z_vel,
        #         roll_vel, pitch_vel, yaw_vel,
        #         x_accel , y_accel  , z_accel]
        
        odom1_differential: true
        odom1_queue_size: 10
        
        imu0: /imu
        imu0_config: [false, false, false,
                      false,  false,  <mark>true</mark>,
                      false, false, false,
                      false,  false,  <mark>true</mark>,
                      <mark>true</mark>, <mark>true</mark>,  false]
        imu0_differential: false
        imu0_relative: true
        imu0_queue_size: 10
</pre>
* Now, you will do an update on parameter file (`ekf_limo.yaml`)  and add remappings to `odom/combined` in `robot_localization/launch/ekf.launch py` as follows :
<pre>
def generate_launch_description():
    return LaunchDescription([
        launch_ros.actions.Node(
            package='robot_localization',
            executable='ekf_node',
            name='ekf_filter_node',
            output='screen',
            <mark>parameters=[os.path.join(get_package_share_directory("robot_localization"), 'params', <mark>'ekf_limo.yaml'</mark>), {'use_sim_time': True}],</mark>
            <mark>remappings=[('odometry/filtered', 'odom/combined')],</mark>
           ),
])
</pre>
* Build and source your workspace then launch the EKF node while `gazebo simulation`, `noisy_odometry` and `laser_scan_matcher` nodes are already working.
```
colcon build
source install/setup.bash
```
```
ros2 launch robot_localization ekf.launch.py
```
* Verify that the `/odometry/combined` topic is being published: `ros2 topic list` and `ros2 topic echo /odometry/combined --field pose.pose.position`
* Visualize the ROS graph using to check whether EKF node subscribes expected topics.
```
ros2 run rqt_graph rqt_graph
```
<p align="center">
<img width="900" height="400" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/ekf_rosgraph.png" />
</p>

Here, you should observe that `/ekf_filter_node` subscribes `/odom/wheel/noisy`, `/odom/filter` and `/imu` to publish filtered odometry on the `/odom/combined` topic. 

### Task 2: Visualize Odometry sources with PlotJuggler

To get better understand the sensor fusion for combined odometry, you will use [PlotJuggler](https://facontidavide.github.io/PlotJuggler/index.html) to visualize and analyze different odomety sources and compare it with ground truth robot's odometry data to verify the success of the EKF fusion algorithm.

* Install and launch PlotJuggler:
```
sudo apt install ros-humble-plotjuggler-ros
```
* Run it through:
```
ros2 run plotjuggler plotjuggler
```
* Set Buffer to at least 150 to record more timestamps and click `Start` to begin adding topics for visualization. 

<p align="center">
<img width="306" height="99" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/juggler.png" />
</p>

* You should see the list of topics available as shown in the figure below. Subscribe to the following odometry topics:.
  * `/odom/combined` - The filtered odometry data after EKF fusion.
  * `/odom/perfect` - The ground-truth position of the robot.
  * `/odom/wheel/noisy` - The raw, noisy odometry data from wheel encoders.
  * `/odom/laser` - The raw, oodometry data from laser scanner.

<p align="center">
<img width="700" height="478" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/juggler_topics.png" />
</p>

* Create an `XY` plot which can be done by dragging and dropping the topic `x` and `y` values using the `RIGHT MOUSE` button not `LEFT MOUSE`.
   * Drag and drop the topic `/odom/combined`  pose→pose→position→ `x` and `y` onto the plotting area.

<p align="center">
<img width="800" height="300" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/juggler_xy_plot.png" />
</p>

   * Repeat this process to add `/odom/perfect` and `/odom/wheel/noisy` and `/odom/laser` for comparison. If you have difficulties please refer to [here](https://facontidavide.github.io/PlotJuggler/visualization_howto/index.html)

### Task 3: Drive the Robot and Visualize in PlotJuggler

Now, you will control the robot and observe the filtered and unfiltered positions and see how the EKF helps correct discrepancies unless the robot experiences wheel slip or collisions.
* Open a new terminal and start the `rqt_robot_steering` graphical interface to control the LIMO robot:
```
sudo apt install ros-humble-rqt-robot-steering
```
```
ros2 run rqt_robot_steering rqt_robot_steering 
```
The `rqt_robot_steering` window allows you to control the robot using a graphical interface. Use the sliders to adjust the robot's linear and angular velocities.

* Drive the robot and observe odometry data in PlotJuggler. 
    * Without sharp turns and hitting obstacle, you should observe that the filtered odometry data `/odom/combined` is closer to the ground truth `/odom` than the noisy wheel odometry data `/odom/wheel/noisy`.

<p align="center">
<img width="900" height="400" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/ekf_plot.png" />
</p>

* Make turns and see how EKF filter corrects these discrepancies using fusion of IMU and encoder data to provide a more accurate position tracking. 
* When the robot collides with an obstacle or experiences wheel slippage, notice how the noisy odometry data may become inaccurate also effects the filtered `/odom/combined` accuracy negatively.
