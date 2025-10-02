The goal of this workshop is to make you familiar with the LIMO robot platform which will be used throughout all remaining workshop sessions. This includes the robot's components and functionality, tools for remote operation, software development and simulation. We are going to use Visual Studio Code (VSC) which will serve as a development platform but also enable remote interaction with the robot. It is a fairly intuitive development environment but you might want to refer to some [docs](https://code.visualstudio.com/docs) if some of the concepts are not very clear.

Moreover, you will start using ROS tools for inspecting your robot sensor data and you will use teleoparation package for navigating your robot around.

# 1. Build and run the ROS2 LIMO Gazebo Simulator
For the LIMO simulator, we are going to use the [ROS2 Humble (https://docs.ros.org/en/humble/index.html) release which runs exclusively on [Ubuntu 22.04.3 LTS](https://releases.ubuntu.com/jammy/). The simulator can either be deployed using a docker container (this is a default option for the lab PCs) or be installed natively on your PC or a virtual machine (e.g. [VMWare Workstation Player](https://www.vmware.com/uk/products/workstation-player.html)), depending on how comfortable you feel with each option and also what PC hardware and OS you have at home. If you struggle with any of the following steps, please ask the staff for help during workshops.

Please ensure that all software updates on your Ubuntu OS are completed before commencing with the rest of the instructions.

* Install robot simulation software and all dependencies.
```
git clone https://github.com/kivrakh/KV6022_limo_ros2

sudo apt-get install -y --no-install-recommends build-essential cmake git python3-pip ros-humble-rmw-cyclonedds-cpp ros-humble-rviz2* ros-humble-teleop-twist-keyboard ros-humble-xacro ros-humble-imu-tools ros-humble-image-* python3-colcon-common-extensions python3-rosdep

sudo rosdep init
rosdep update
rosdep install --from-paths src -y --ignore-src
```

* Build and Run the simulator and check that everything is running as expected
```
cd KV6022_limo_ros2 && ./build.sh
source install/setup.bash
ros2 launch limo_gazebosim limo_gazebo_diff.launch.py
```
# 2. Basic operations
1. Inspect the robot's nodes and topics by using the ros2 node and ros2 topic commands (in a new terminal, no need to source this time). When you type the command without any additional arguments, you should see all available options. Display and compare the format of the following topics /odom, /scan, /tf and /camera/color/image_raw. You might want to also refer to the official [node](https://docs.ros.org/en/foxy/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Nodes/Understanding-ROS2-Nodes.html) and [topic](https://docs.ros.org/en/foxy/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Topics/Understanding-ROS2-Topics.html) tutorials.

2. Now, let us use the graphical visualiser [RVIZ](https://github.com/ros2/rviz) to look at the robot and its sensor topics. Start by typing rviz2 -d src/limo_description/rviz/model_sensors_real.rviz which uses a pre-defined configuration file, and you should see the interface with a robot model and its sensor data displayed in the robot's touch screen (and your VNC screen too!). To get familiar with the interface, adjust the laser scan visualisation options and see how these affect the output. Try to add new visualisation for sensors not included in the provided configuration (e.g. odometry).

3. Teleoperation. Leave the RVIZ running. In a new terminal start the keyboard teleoperation node `ros2 run teleop_twist_keyboard teleop_twist_keyboard` and drive the robot around using the keyboard. Have fun, but pay attention to other robots and your human fellows, though!

4. Let's now send some basic robot control commands using ROS topics. The robot's speed can be controlled by the /cmd_vel topic. Use the ros2 topic pub functionality to send a single Twist message (linear and angular velocity command) as in this example:

```
ros2 topic pub --once /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.0, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.5}}"
```
Now, adjust the linear components of the Twist message and see the resulting trajectory.

5. Using your knowledge of the topic publishing, issue a series of commands that will drive the robot:

    * in a circle with a radius of 0.5 m;
    * in a 1 m square.

Try using as few commands as possible.

