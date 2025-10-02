The goal of this workshop is to make you familiar with the LIMO robot platform which will be used throughout all remaining workshop sessions. This includes the robot's components and functionality, tools for remote operation, software development and simulation. We are going to use Visual Studio Code (VSC) which will serve as a development platform but also enable remote interaction with the robot. It is a fairly intuitive development environment but you might want to refer to some [docs](https://code.visualstudio.com/docs) if some of the concepts are not very clear.

Moreover, you will start using ROS tools for inspecting your robot sensor data and you will use teleoparation package for navigating your robot around.

# 1. Build and run the ROS2 LIMO Simulator
For the LIMO simulator, we are going to use the [ROS2 Humble (https://docs.ros.org/en/humble/index.html) release which runs exclusively on [Ubuntu 22.04.3 LTS](https://releases.ubuntu.com/jammy/). The simulator can either be deployed using a docker container (this is a default option for the lab PCs) or be installed natively on your PC or a virtual machine (e.g. [VMWare Workstation Player](https://www.vmware.com/uk/products/workstation-player.html)), depending on how comfortable you feel with each option and also what PC hardware and OS you have at home. If you struggle with any of the following steps, please ask the staff for help during workshops.

Please ensure that all software updates on your Ubuntu OS are completed before commencing with the rest of the instructions.

* Install robot simulation software and all dependencies.
```
git clone https://github.com/kivrakh/KV6022_limo_ros2

sudo apt-get install -y --no-install-recommends build-essential cmake git python3-pip ros-humble-rmw-cyclonedds-cpp ros-humble-rviz2* ros-humble-teleop-twist-keyboard ros-humble-xacro ros-humble-imu-tools ros-humble-image-* python3-colcon-common-extensions python3-rosdep

sudo rosdep init
rosdep update
rosdep install --from-paths src -y --ignore-src

cd KV6022_limo_ros2 && ./build.sh
```

* Run the simulator and check that everything is running as expected
```
source install/setup.bash
ros2 launch limo_gazebosim limo_gazebo_diff.launch.py
```

