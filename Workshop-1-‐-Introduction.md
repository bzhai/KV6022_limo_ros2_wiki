The goal of this workshop is to make you familiar with the LIMO robot platform which will be used throughout all remaining workshop sessions. This includes the robot's components and functionality, tools for remote operation, software development and simulation. We are going to use Visual Studio Code (VSC) which will serve as a development platform but also enable remote interaction with the robot. It is a fairly intuitive development environment but you might want to refer to some [docs](https://code.visualstudio.com/docs) if some of the concepts are not very clear.

Moreover, you will start using ROS tools for inspecting your robot sensor data and you will use teleoparation package for navigating your robot around.

# 1. Install Terminator
Install terminator (enabling the management of several terminals in a single window.) using debian package:
```
sudo apt install terminator
```
To open the terminator:
* click on the terminal icon from the startup
* choose to Split Horizontally or Split Vertically by right-clicking a terminal window to have multiple terminal windows
* Try to achieve three terminal windows that look like so
![alt](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/terminator.png)

# 2. Using Linux Terminal

* Type `mkdir -p ros2_ws/src`, then `ls`
You will see there is a new folder `ros2_ws` and a `src` folder inside `ros2_ws`. 
Directories, like `ros2_ws`, are colored in blue.
* Enter `pwd` into the terminal
This will show you the full path of the directory you are working in.
* Enter `cd ros2_ws` into the terminal.
The prompt should change to directory to `ros2_ws`
Typing `pwd` will show you now in the `ros2_ws` directory
* Enter `cd ..` into the terminal  (`..` is the parent folder.)
The prompt should therefore indicate the parent folder `home`
* Type `touch ~/ros2_ws/src/hello_world.py && cd ~/ros2_ws/src`
This will create a new document named “hello_world.py” under ros_ws/src directory
* Type `mv hello_world.py hello_world2.py`, followed by `ls`.
You will notice that the file has been renamed to `hello_world2.py`.
This step shows how `mv` can rename or move files to directories.
* Type `cp hello_world2.py ~/ros2_ws/hello_world2.py`, then `ls ~/ros2_ws`
You will see that test.txt has been copied to test copy.txt
* Type `rm hello_world2.py`, then `ls`
You will notice that `hello_world2.py` is no longer there.


# 3. [Using turtlesim](https://docs.ros.org/en/humble/Tutorials/Beginner-CLI-Tools/Introducing-Turtlesim/Introducing-Turtlesim.html)

Turtlesim is a lightweight simulator for learning ROS 2. It illustrates what ROS 2 does at the most basic level to give you an idea of what you will do with a real robot or a robot simulation later on.

This tutorial touches upon core ROS 2 concepts, like nodes, topics, and services. All of these concepts will be elaborated on in later tutorials; for now, you will simply set up the tools and get a feel for them.

* Install the turtlesim package for your ROS 2 Humble:
```
sudo apt update
sudo apt install ros-humble-turtlesim
```
* Start turtlesim
```
ros2 run turtlesim turtlesim_node
```
Under the command, you will see messages from the node. There you can see the default turtle’s name and the coordinates where it spawns.

The simulator window should appear, with a random turtle in the center.
![alt](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/turtlesim.png)

*Use turtlesim
Open a new terminal and source ROS 2 again.

Now you will run a new node to control the turtle in the first node:
```
ros2 run turtlesim turtle_teleop_key
```
At this point you should have three windows open: a terminal running `turtlesim_node`, a terminal running `turtle_teleop_key` and the turtlesim window. Arrange these windows so that you can see the turtlesim window, but also have the terminal running `turtle_teleop_key` active so that you can control the turtle in turtlesim.

Use the arrow keys on your keyboard to control the turtle. It will move around the screen, using its attached “pen” to draw the path it followed so far.

> [!NOTE]
> Pressing an arrow key will only cause the turtle to move a short distance and then stop. This is because, realistically, you wouldn’t want a robot to continue carrying on an instruction if, for example, the operator lost the connection to the robot.

You can see the nodes, and their associated topics, services, and actions, using the `list` subcommands of the respective commands:
```
ros2 node list
ros2 topic list
ros2 service list
ros2 action list
```

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
![alt](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/gazebo_limo.jpg)
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

# 3. Additional tasks

* Learn more about the [nodes](https://docs.ros.org/en/foxy/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Nodes/Understanding-ROS2-Nodes.html) and [topics](https://docs.ros.org/en/humble/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Topics/Understanding-ROS2-Topics.html). Whilst these concepts will be covered later in the course, this will give you a first glimpse into various ROS functionality associated with LIMO.
* Learn how to create a simple ROS2 [publisher node](https://docs.ros.org/en/humble/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Py-Publisher-And-Subscriber.html) and modify the example such that it sends a single /cmd_vel command to control LIMO from the script.

