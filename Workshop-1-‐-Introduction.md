The goal of this workshop is to make you familiar with the ROS 2 environment and LIMO robot simulation platform which will be used throughout all remaining workshop sessions. This includes the robot's components and functionality, tools for remote operation, software development and simulation. 

We are going to use Visual Studio Code (VSC) which will serve as a development platform but also enable remote interaction with the robot. It is a fairly intuitive development environment but you might want to refer to some [docs](https://code.visualstudio.com/docs) and [installation](https://code.visualstudio.com/download) if some of the concepts are not very clear.

**Learning objectives**
1. Launch and manage multiple terminals efficiently. Practice essential Linux commands in a ROS 2 workspace layout.
1. Run ROS 2 nodes (turtlesim), list nodes/topics/services/actions and publish messages.
3. Build and launch the LIMO Gazebo simulator, visualize nodes and topics.
4. Teleoperate a robot and publish `/cmd_vel` commands programmatically.

# 1. Install Terminator
Terminator makes multiple terminals in one window. 
* Install terminator using debian package:
```
sudo apt install terminator
```
* Open Terminator:
   * Click on the terminal icon from the startup
   * Choose to "Split Horizontally" or "Split Vertically" by right-clicking a terminal window to have multiple terminal windows
* Try to achieve three terminal windows (1x2, 2x2) that look like so
<img src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/terminator.png" width="1000">

# 2. Linux Terminal Warm-Up
Practice essential commands while creating a simple ROS 2 workspace.

* Create a workspace (directory):
```
mkdir -p ~/ros2_ws/src
ls
```
You will see there is a blue `ros2_ws` directory and a `src` folder inside it. 
* Where am I?
```
pwd
```
This will show you the full path of the directory you are working in.
* Move into the workspace:
```
cd ~/ros2_ws
pwd
```
The prompt should change to the directory `ros2_ws`
* Go up one level
```
cd ..
```
The prompt should therefore indicate the parent folder `home`
* Create a file and move into `src`:
```
touch ~/ros2_ws/src/hello_world.py && cd ~/ros2_ws/src
ls
```
This will create a new document named `hello_world.py` under `ros_ws/src` directory
* Rename a file:
```
mv hello_world.py hello_world2.py
ls
```
You will notice that the file has been renamed to `hello_world2.py`.
This step shows how `mv` can rename or move files to directories.
* Copy a file: 
```
cp hello_world2.py ~/ros2_ws/hello_world2.py
ls ~/ros2_ws
```
You should now see `hello_world2.py` in the workspace root.
* Remove a file:
```
rm hello_world2.py
ls
```
You will notice that `hello_world2.py` is no longer there.


# 3. First ROS 2 App: Turtlesim

> [!IMPORTANT] 
>## Quick fix on ROS 2 Networking
>If we wanted to avoid other devices' nodes, we can use the `ROS_LOCALHOST_ONLY` environment variable to limit communication to only nodes >on the same device. To keep communications on your own machine only (Local host only variable), set by adding it to your `~/.bashrc` and the daemon (discovery of nodes) must be restarted for changes in the environment to be propagated:
>```
>echo 'export ROS_LOCALHOST_ONLY=1' >> ~/.bashrc
>source ~/.bashrc
>ros2 daemon stop
>ros2 daemon start
>```

Turtlesim is a lightweight 2D simulator or graphical user interface for learning core ROS 2 concepts, like nodes, topics, and services. It illustrates what ROS 2 does at the most basic level to give you an idea of what you will do with a real robot or a robot simulation later on. All of these concepts will be elaborated on in later workshops.

* Install the `turtlesim` package for your ROS 2 Humble:
```
sudo apt update
sudo apt install ros-humble-turtlesim
```
* Launch turtlesim simulator
```
ros2 run turtlesim turtlesim_node
```
Under the command, you will see messages from the node. There you can see the default turtle’s name and the coordinates where it spawns.

The simulator window should appear, with a random turtle in the center.

![alt](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/turtlesim.png)

* Use turtlesim

Open or use a new terminal. Now you will run a new node (`turtle_teleop_key`) to control the turtle in the first node:
```
ros2 run turtlesim turtle_teleop_key
```
At this point you should have three windows open: a terminal running `turtlesim_node`, a terminal running `turtle_teleop_key` and the `turtlesim` window. Arrange these windows so that you can see the turtlesim window, but also have the terminal running `turtle_teleop_key` active so that you can control the turtle in turtlesim.

Keep the teleop terminal focused and use arrow keys on your keyboard to control/move the turtle. It will move around the screen, using its attached "pen" to draw the path it followed so far.

> [!NOTE]
> Pressing an arrow key will only cause the turtle to move a short distance and then stop. This is because, realistically, you wouldn’t want a robot to continue carrying on an instruction if, for example, the operator lost the connection to the robot.

* In a another terminal , explore the ROS 2 nodes, and their associated topics, services, and actions, using the `list` subcommands of the respective commands:
```
ros2 node list
```
```
ros2 topic list
```
```
ros2 service list
```
```
ros2 action list
```

# 4. Build and run the ROS 2 LIMO Gazebo Simulator

* Install robot simulation software and all dependencies from the module repo.
```
git clone https://github.com/kivrakh/KV6022_limo_ros2.git

sudo apt-get install -y --no-install-recommends build-essential cmake git python3-pip ros-humble-rmw-cyclonedds-cpp ros-humble-rviz2* ros-humble-teleop-twist-keyboard ros-humble-xacro ros-humble-imu-tools ros-humble-image-* python3-colcon-common-extensions python3-rosdep

sudo rosdep init
rosdep update
cd ~/KV6022_limo_ros2
rosdep install --from-paths src -y --ignore-src
```

* Build and Source Gazebo simulator
```
colcon build --symlink-install # run inside the repo (KV6022_limo_ros2)
source install/setup.bash
```
* Run Gazebo simulator and check that everything is running as expected
```
ros2 launch limo_gazebosim limo_gazebo_diff.launch.py
```
You should see Gazebo driving area world with the LIMO robot:
![alt](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/gazebo_limo.jpg)


# 5. Basic ROS 2 operations
1. Inspect nodes & topics: Inspect the robot's nodes and topics by using the `ros2 node` and `ros2 topic` commands. When you type the command without any additional arguments, you should see all available options. Display and compare the format of the following topics `/odom`, `/scan`, `/tf` and `/depth_camera/color/image_raw`. You might also want to refer to the official [node](https://docs.ros.org/en/foxy/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Nodes/Understanding-ROS2-Nodes.html) and [topic](https://docs.ros.org/en/foxy/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Topics/Understanding-ROS2-Topics.html) tutorials.

<!-- 2. Now, let us use the graphical visualiser [RVIZ](https://github.com/ros2/rviz) to look at the robot and its sensor topics. Start by typing rviz2 -d src/limo_description/rviz/model_sensors_real.rviz which uses a pre-defined configuration file, and you should see the interface with a robot model and its sensor data displayed in the robot's touch screen (and your VNC screen too!). To get familiar with the interface, adjust the laser scan visualisation options and see how these affect the output. Try to add new visualisation for sensors not included in the provided configuration (e.g. odometry). -->

2. Teleoperate
In a new terminal start the keyboard teleoperation node `ros2 run teleop_twist_keyboard teleop_twist_keyboard` and drive the robot around using the keyboard. Use the on-screen key hints to drive.

3. Publish velocity commands. Let's now send some basic robot control commands using ROS topics. The robot's speed can be controlled by the `/cmd_vel` topic. Use the `ros2 topic pub` functionality to send a single Twist message (linear and angular velocity command) as in this example:
```
ros2 topic pub --once /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.0, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.5}}"
```
Now, adjust the `linear.x` and `angular.z` components of the Twist message and see the resulting trajectory.

4. Using your knowledge of the topic publishing, issue a series of commands that will drive the robot:
    * in a circle with a radius of 0.5 m
        * Hint: v=𝜔⋅r. Choose linear.x = 0.25, angular.z = 0.5;
    * in a 1 m square, stop, rotate ~90°, repeat.

Try using as few commands as possible. Publish a short repeating twist (hint: omit `--once` and use `--rate`)

`ros2 topic pub --rate 5 /cmd_vel geometry_msgs/msg/Twist '{...}'`

Use `Ctrl-C` to stop publishing.

# 6. Additional tasks

* Learn more about the [nodes](https://docs.ros.org/en/foxy/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Nodes/Understanding-ROS2-Nodes.html) and [topics](https://docs.ros.org/en/humble/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Topics/Understanding-ROS2-Topics.html). Whilst these concepts will be covered later in the module, this will give you a first glimpse into various ROS functionality associated with LIMO.
* Learn how to create a simple ROS2 [publisher node](https://docs.ros.org/en/humble/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Py-Publisher-And-Subscriber.html) and modify the example such that it sends a single `/cmd_vel` command to control LIMO from the script.

