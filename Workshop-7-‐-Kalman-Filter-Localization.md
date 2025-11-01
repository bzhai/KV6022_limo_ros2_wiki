### Overview
In this lab, you will gain practical experience using an Extended Kalman Filter (EKF) for robot localization with the `robot_localization` package. By completing this workshop, you will experiment with fusing noisy wheel encoder odometry data, laser and IMU sensor data and observing how sensor fusion can improve localization accuracy in scenarios involving wheel slip, turns, obstacle collision and other disturbances.

### Preparation 1: Setting wheel encoder sensor
You need modify the LIMO robot gazebo configuration to use wheel encoder odometry as the source for odometry information.
* Open the LIMO robot gazebo configuration file [limo_diff.gazebo](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/limo_description/urdf/limo_diff/limo_diff.gazebo):
```
sudo gedit ~/KV6022_limo_ros2/src/limo_description/urdf/limo_diff/limo_diff.gazebo
```
In the `libgazebo_ros_diff_drive.so` plugin set the `odometry_source` parameter to `0` to use odometry source using the wheel encoder sensor: 
<pre>
&lt;plugin name="four_diff_controller" filename="libgazebo_ros_diff_drive.so"&gt;
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
    &lt;publish_odom_tf&gt;true&lt;/publish_odom_tf&gt;
    &lt;publish_wheel_tf&gt;false&lt;/publish_wheel_tf&gt;
    &lt;odometry_topic&gt;odom&lt;/odometry_topic&gt;
    &lt;odometry_frame&gt;odom&lt;/odometry_frame&gt;
    &lt;robot_base_frame&gt;base_footprint&lt;/robot_base_frame&gt;
    &lt;!-- Odometry source, 0 for ENCODER, 1 for WORLD, defaults to WORLD --&gt;
    <mark>&lt;odometry_source&gt;0&lt;/odometry_source&gt;
    &lt;ros&gt;
        &lt;remapping&gt;odom:=/odom/wheel&lt;/remapping&gt;
    &lt;/ros&gt;</mark>
&lt;/plugin&gt;
</pre>
* To get the robot's actual (ground truth) position to compare the EKF estimation. Copy paste the contents into following of the above description file. It should publish the ground truth to `odom/perfect`
<pre>
&lt;gazebo&gt;
    &lt;plugin name="limo_diff_drive_perfect" filename="libgazebo_ros_p3d.so"&gt;
        &lt;always_on&gt;true&lt;/always_on&gt;
        &lt;update_rate&gt;30.0&lt;/update_rate&gt;
        &lt;body_name&gt;base_link&lt;/body_name&gt;
        &lt;topic_name&gt;odom/perfect&lt;/topic_name&gt;
        &lt;gaussian_noise&gt;0.01&lt;/gaussian_noise&gt;
        &lt;frame_name&gt;world&lt;/frame_name&gt;
        &lt;xyz_offset&gt;0 0 0&lt;/xyz_offset&gt;
        &lt;rpy_offset&gt;0 0 0&lt;/rpy_offset&gt;
        &lt;ros&gt;
           <mark>&lt;remapping&gt;odom:=/odom/perfect&lt;/remapping&gt;</mark>
        &lt;/ros&gt;</mark>
    &lt;/plugin&gt;
&lt;/gazebo&gt;
</pre>
* Build and source your workspace
* Launch the Gazebo simulation
* Verify the wheel encoder odometry is in  place. Put an firm object in front of robot such as `Cinder Block` from gazebo models. Robot is towards to `-y` direction, that is we expect reducing y value when robot moves in forward. Observe that even robot is not moving the y position of the robot keep increasing due to fact that robot is considering it is moving forward according wheels turns.
```
ros2 topic echo /odom --field pose.pose.position
```
CINDER_BLOCK
### Preparation 2: Adding noise to the wheel odometry model
* You will add to the odometry data to better reflect real-world scenario as Gazebo provides near-perfect sensory model. Review [noisy_odometry.py]() which will add gaussion noise to odometry motion model explained [here](https://blog.lxsang.me/post/id/16).
* Inspect IMU sensor model in the LIMO's model. Search for the `<link name="imu_link"` tag where you will see the `<noise>` tag under the `<angular_velocity>` and `<linear_acceleration>` tags. IMU noise is already configured.
### Preparation 3: Setting up Sensor Fusion Package
You will now integrate and configure the `robot_localization` [package](http://docs.ros.org/en/noetic/api/robot_localization/html/index.html) to fuse noisy wheel odometry and IMU data using an EKF to achieve better localization.
* Clone the `robot_localization` package repository (humble branch) into your `KV6022_limo_ros2` workspace:
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
* Configure the EKF node by creating the following configuration file. Save the following configuration as a `ekf_limo.yaml` file to under the `robot_localization` package `params` directory. If you want to learn the explanations of each parameter, refer to `ekf.yaml` in the same directory:
<pre>
ekf_filter_node:
  ros__parameters:
    frequency: 30.0
    two_d_mode: true

    odom_frame: odom
    base_link_frame: base_link
    world_frame: odom

    <mark>odom0: /odom/wheel/noisy</mark>
    odom0_config: [false, false, false,
                   false, false, false,
                   <mark>true</mark>,  <mark>true</mark>,  <mark>true</mark>,
                   false, false, <mark>true</mark>,
                   false, false, false]
    odom0_differential: false
    odom0_queue_size: 10
    
    <mark>imu0: /imu</mark>
    imu0_config: [false, false, false,
                  <mark>true</mark>,  <mark>true</mark>,  <mark>true</mark>,
                  false, false, false,
                  <mark>true</mark>,  <mark>true</mark>,  <mark>true</mark>,
                  false,  false,  false]
    imu0_differential: true
    imu0_queue_size: 10
</pre>
* Update parameter file to `ekf2_limo.yaml` in `robot_localization/launch/ekf.launch py` and add remappings to `odom/combined`:
<pre>
def generate_launch_description():
    return LaunchDescription([
        launch_ros.actions.Node(
            package='robot_localization',
            executable='ekf_node',
            name='ekf_filter_node',
            output='screen',
            <mark>parameters=[os.path.join(get_package_share_directory("robot_localization"), 'params', 'ekf_limo.yaml'), {'use_sim_time': True}],</mark>
            <mark>remappings=[('odometry/filtered', 'odom/combined')],</mark>
           ),
])
</pre>
* Launch the EKF node (make sure simulation and noisy_odometry node is working)
```
colcon build
source install/setup.bash
ros2 launch robot_localization ekf.launch.py
```
* Verify that the `/odometry/combined` topic is being published: `ros2 topic list`
* Visualize the ROS graph using `rqt_graph`:
```
ros2 run rqt_graph rqt_graph
```
img/graph.png

Here, you should observe that `/ekf_filter_node` subscribes `/odom/wheel/noisy` and `/imu` and publish filtered odometry on the `/odom/combined` topic. This topic is then subscribed by `/plotjuggler` node to visualize it.

Task 1: Visualize with PlotJuggler

In this task, you will use [PlotJuggler](https://facontidavide.github.io/PlotJuggler/index.html) to visualize and analyze noisy, filtered and ground truth robot's odometry data to verify the success of the EKF fusion algorithm.

* Install and launch PlotJuggler:
```
sudo apt install ros-humble-plotjuggler-ros
```
```
ros2 run plotjuggler plotjuggler
```
* Set Buffer to at least 150 to record more timestamps and Click Start to begin adding topics for visualization. 
img/juggler.png
* Subscribe to the following odometry topics:.
  * `/odom/combined` - The filtered odometry data after EKF fusion.
  * `/odom` - The ground-truth position of the robot.
  * `/odom/wheel/noisy` - The raw, noisy odometry data from wheel encoders.

You should see the list of topics available as shown in the figure below
img/topics.png

* Create an `XY` plot to compare the noisy, filtered, and ground-truth positions of the robot. To activate this mode, drag and drop the curve that shall be used as X axis using the `RIGHT MOUSE` button instead of the `LEFT` one. 
   * Drag and drop the topic `/odom/combined`  pose→pose→position→ X and Y onto the plotting area.
img/plot.png
   * Repeat this process to add `/odom/` and `/odom/wheel/noisy` for comparison.

If you have difficulties  please refer to [here](https://facontidavide.github.io/PlotJuggler/visualization_howto/index.html)

### Task 2: Drive the Robot and Visualize in PlotJuggler

Now, you will control the robot and observe the filtered and unfiltered positions and see how the EKF helps correct discrepancies unless the robot experiences wheel slip or collisions.
* Open a new terminal and start the `rqt_robot_steering` graphical interface to control the LIMO robot:
```
sudo apt install ros-humble-rqt-robot-steering
rqt_robot_steering
```
The `rqt_robot_steering` window allows you to control the robot using a graphical interface. Use the sliders to adjust the robot's linear and angular velocities.

* Drive the robot and observe odometry data in PlotJuggler. 
    * Without sharp turns and hitting obstacle, you should observe that the filtered odometry data (`/odom/combined`) is closer to the ground truth (`/odom`) than the noisy data (`/odom/wheel/noisy`).
      
     img/result0.png

     * Make turns and see how EKF filter corrects these discrepancies using fusion of IMU and encoder data to provide a more accurate position. 
     * When the robot collides with an obstacle or experiences wheel slippage, notice how the noisy odometry data may become inaccurate also effects the filtered (`/odom/combined`) accuracy negatively.
  img/result-arrow.png
        
ToDO:
 * Wheel encoder performance can be evaluated by playing with the Gazebo friction parameter
