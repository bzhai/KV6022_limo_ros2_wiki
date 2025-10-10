The goal of this workshop is to make you familiar with the LIMO robot platform with the different locomotion capabilities and ROS 2 Python packages and nodes. You will inspect simulated LIMO nodes/topics, visualize in RViz, teleoperate the robot and write a Python controller that publishes velocity commands for navigating your robot.

<!-- Moreover, you will start using ROS tools for inspecting your robot sensor data and you will use teleoparation package for navigating your robot around. -->

> [!IMPORTANT] 
>## Quick fix on ROS 2 Networking
>If we wanted to avoid other devices' nodes, we can use the `ROS_LOCALHOST_ONLY` environment variable to limit communication to only nodes on the same device. To keep communications on your own machine only (Local host only variable), set by adding it to your `~/.bashrc` and the daemon (discovery of nodes) must be restarted for changes in the environment to be propagated:
>```
>echo 'export ROS_LOCALHOST_ONLY=1' >> ~/.bashrc
>source ~/.bashrc
>ros2 daemon stop
>ros2 daemon start
>```
## 1. Get to know your LIMO ROS 2 nodes and topics 

Launch the differential-drive LIMO in Gazebo and find out the following information using ROS 2 Command Line Interface (CLI):

* How many topics and nodes does the LIMO robot have?
* What topic is used to move the LIMO robot? (usually `/cmd_vel`) 
  * Check publishers/subscribers and message type.
* What topic provides the current pose (position and orientation) of the robot? (usually `/odom`) 
   * How frequently is the pose being published?
   * What message type does this topic use?
   * Display current pose topic messages
       * What are the key fields in the message (e.g., `x`, `y`, `theta`)?
* Display the ROS 2 computation graph (`rqt_graph`) and analyze the node relationships.

* Now, let us use the graphical visualiser [RVIZ](https://github.com/ros2/rviz) to look at the robot and its sensor topics. In another terminal window open `rviz2` with the following pre-defined configuration file `KV6022_limo_ros2/src/limo_gazebosim/rviz`. You should see the interface with a robot model and its sensor data using the command:
```
rviz2 -d ~/KV6022_limo_ros2/src/limo_gazebosim/rviz/urdf.rviz
```
* To get familiar with the RViz 2 interface, adjust the laser scan visualisation options and see how these affect the output. Try to add new visualisation for sensors not included in the provided configuration (e.g. odometry).

## 2. Teleoperate LIMO with Differential, Ackerman and Mecanum steering
You will drive any steering mode via `/cmd_vel` using the keyboard teleop.
* First, start with a differential drive sterring
```
ros2 launch limo_gazebosim limo_gazebo_diff.launch.py
```
* Next, try Ackermann steering mode
```
ros2 launch limo_gazebosim limo_gazebo_ackerman_drive.launch.py
```
* Finally, experiment Mecanum drive
```
ros2 launch limo_gazebosim limo_gazebo_mecanum_drive.launch.py
```
## 3. Installing and Running Visual Studio Code (Vscode) 
There are many ways to manage and edit your Python code but we will use VS Code. Install it using the following command:
```
cd ~
wget -O code_1.104.3-1759409451_amd64.deb https://code.visualstudio.com/sha/download?build=stable&os=linux-deb-x64
sudo dpkg -i code_1.104.3-1759409451_amd64.deb
```
Once installed, open the extensions tab `CTRL+SHIFT+X` and search for `ROS`. Install the `Microsoft ROS` extension.

You can then select `File > Open Folder`. Navigate to your ROS 2 workspace (e.g., `KV6022_limo_ros2/src`) and open the folder. You can now use VS Code to manage and edit ROS packages.

## 4. Create a new ROS 2 package and Build
A single workspace can contain as many packages as you want, each in its own folder. Best practice is to have a `src` folder within your workspace and to create your packages in there. Observe that under `KV6022_limo_ros2/src` workspace, there exist `limo_description`, `limo_gazebosim` and `limo_msgs` packages.

* So, navigate into `KV6022_limo_ros2/src` and decide on a name for your package and run the package creation command to keep all your work in it (e.g, `Week2_lab`) - you may want to follow the [official instructions](https://docs.ros.org/en/humble/Tutorials/Beginner-Client-Libraries/Creating-Your-First-ROS2-Package.html).

* Return to the root of your workspace (`KV6022_limo_ros2/`) and build the workspace that now has this empty package and others using `colcon build`.

> [!NOTE]
>`colcon` will generate its output wherever it's run. Make sure you're in the workspace root before building.

## 5. Your first ROS controller (Creating a node with a publisher)
You can control your LIMO robot autonomously by writing a small controller in Python that publishes velocity commands on a dedicated topic. To do so, you can modify the `publisher.py` script (seen during the demonstration) below to send robot control commands.

To do so, remember you need to change the `topic` on which to publish from topic to `cmd_vel`, the published message from `String` to `geometry_msgs/msg/Twist` and correctly populate the message's fields.
```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String

class MinimalPublisher(Node):

    def __init__(self):
        super().__init__('minimal_publisher')
        self.publisher_ = self.create_publisher(String, 'topic', 10)
        timer_period = 0.5  # seconds
        self.timer = self.create_timer(timer_period, self.timer_callback)
        self.i = 0

    def timer_callback(self):
        msg = String()
        msg.data = 'Hello World: %d' % self.i
        self.publisher_.publish(msg)
        self.get_logger().info('Publishing: "%s"' % msg.data)
        self.i += 1

def main(args=None):
    rclpy.init(args=args)

    minimal_publisher = MinimalPublisher()

    rclpy.spin(minimal_publisher)

    # Destroy the node explicitly
    # (optional - otherwise it will be done automatically
    # when the garbage collector destroys the node object)
    minimal_publisher.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

### Running a node
You can run your node in two ways:
1. Direct Python (run the script yourself)
2. ROS 2 CLI (`ros2 run`)

If you want to execute the script without `ros2 run`, make it executable and call it:
```
chmod +x ~/KV6022_limo_ros2/src/Week2_lab/Week2_lab/command_publisher.py
```
```
python3 ~/KV6022_limo_ros2/src/Week2_lab/Week2_lab/command_publisher.py
```
To use with `ros2 run`, we need an additional step to make it deployable in a place where ros2 run can find it.
<!-- 1. Modify `package.xml` with any additional runtime dependencies.
```
<exec_depend>rclpy</exec_depend>
<exec_depend>geometry_msgs</exec_depend>
```
-->
1. Modify the `setup.py` file. To do so, we modify the `console_scripts key` in the `entry_points` dictionary to have our new node in a specific format (The name of the node when calling it through ros2 run = The name of the package.The name of the script, without the `.py` extension. The function, within the script, that will be called. In general, `main`)
<pre>
from setuptools import find_packages, setup
package_name = 'Week2_lab'

setup(
    name=package_name,
    version='0.0.0',
    packages=find_packages(exclude=['test']),
    data_files=[
        ('share/ament_index/resource_index/packages',
         ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
    ],
    install_requires=['setuptools'],
    zip_safe=True,
    maintainer='murilo',
    maintainer_email='aaa@northumbria.ac.uk',
    description='TODO: Package description',
    license='TODO: License declaration',
    tests_require=['pytest'],
    entry_points={
        'console_scripts': [
            <mark>'command_python_publisher = Week2_lab.command_publisher:main'</mark>
        ],
    },
)
</pre>
2. Build and source 
```
cd ~/KV6022_limo_ros2/
colcon build
source install/setup.bash
```
And, with that, we can run `ros2 run my_package my_node`
```
ros2 run Week2_lab command_publisher
```
> [!TIP] 
> If ROS2 is unable to find the node, but it can find the package, then you can rely on `ros2 pkg executables`. For instance, you can run as >follows. If the command outputs nothing, this means that no nodes were found.
>```
>ros2 pkg executables python_package
>```
>The command, at this stage, should output the following.
>```
>Week2_lab command_publisher
>```

## Extension Tasks
* Adapt your controller for LIMO differential and mecanum mode by publishing to `/cmd_vel` accordingly.
* Drive the robot to move in a circle
* Draw a square. Go straight, stop, turn 90°. Repeat it four times.