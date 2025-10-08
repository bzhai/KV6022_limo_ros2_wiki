The goal of this workshop is to make you familiar with the LIMO robot platform with the different locomotion capabilities and ROS 2 Python packages and nodes. You will start using ROS tools for inspecting simulated LIMO robot available nodes and topics (e.g., pose, sensor data) and you will write a simple controller for navigating your robot around.

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

Run the differential drive LIMO robot gazebo simulation and find out the following informations using ROS 2 Command Line Interface (CLI):

* How many topics and nodes does the LIMO robot have?
* What topic is used to move the LIMO robot? How can we use velocity commands to control its movement?
  * Are there any publishers or subscribers to this topic?
* How can we know (what topic is provided) the current pose (position and orientation) of the robot?
   * How frequently is the pose being published?
   * What message type does this topic use?
   * Display current pose topic messages
       * What are the key fields in the message (e.g., x, y, theta)?
* Display the ROS 2 computation graph (`rqt_graph`) and analyze the node relationships.

* Now, let us use the graphical visualiser [RVIZ](https://github.com/ros2/rviz) to look at the robot and its sensor topics. In another terminal window open `rviz2` with the following pre-defined configuration file `KV6022_limo_ros2/src/limo_gazebosim/rviz`. You should see the interface with a robot model and its sensor data using the command:
```
rviz2 -d ~/KV6022_limo_ros2/src/limo_gazebosim/rviz/urdf.rviz
```
* To get familiar with the interface, adjust the laser scan visualisation options and see how these affect the output. Try to add new visualisation for sensors not included in the provided configuration (e.g. odometry).

## 2. Teleoperate Ackerman steering, mecanum steering of LIMO robot
WILL BE ADDED

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
A single workspace can contain as many packages as you want, each in their own folder. Best practice is to have a `src` folder within your workspace, and to create your packages in there. Observe that under `KV6022_limo_ros2/src` workspace there exists `limo_description`, `limo_gazebosim` and `limo_msgs` packages.
* So, navigate into `KV6022_limo_ros2/src` and decide on a name for your package and run the package creation command to keep all your work in it (e.g, `Week2_lab`) - you may want to follow the [official instructions](https://docs.ros.org/en/humble/Tutorials/Beginner-Client-Libraries/Creating-Your-First-ROS2-Package.html).

<!-- Make sure you are in the `src` folder before running the package creation command.

```
cd ~/ros2_ws/src
```
The command syntax for creating a new Python package in ROS 2 is:
```
ros2 pkg create --build-type ament_python <package_name>
```
You will now have a new folder within your workspace's `src` directory called `my_package`. Navigate to the package directory and ensure the structure includes a `setup.py` and `package.xml` file. You will modify these files later to add the nodes you create.
-->

* Return to the root of your workspace (`KV6022_limo_ros2/`) and build the workspace that now has this empty package and others using `colcon`.

<!--- ```
cd ~/ros2_ws
```
> [!NOTE]
>`colcon` will generate its output wherever it's run. Make sure you're in the workspace root before building.
Now you can build the workspace that now has this empty package and others using `colcon`:
```
colcon build
```
-->
## 5. Your first ROS controller (Creating nodes with publisher)
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
<!--
## Running a node
> [!NOTE]
> Modify package.xml with any additional dependencies.
> Modify the setup.py file.
> Build and source

we need an additional step to make it deployable in a place where ros2 run can find it.

Add this node to `setup.py` for execution. Modify `setup.py` as follows:

<pre>
from setuptools import find_packages, setup

package_name = 'python_package_with_a_node'

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
    maintainer_email='murilomarinho@ieee.org',
    description='TODO: Package description',
    license='TODO: License declaration',
    tests_require=['pytest'],
    entry_points={
        'console_scripts': [
            'sample_python_node = python_package_with_a_node.sample_python_node:main',
            <mark>'print_forever_node = python_package_with_a_node.print_forever_node:main'</mark>
        ],
    },
)
</pre>


the example node can be executed with `ros2 run`.
To run the executable you created enter the command:
```
ros2 run my_package my_node
```
> [!NOTE]
> If ROS2 is unable to find the node, but it is able to find the package, then you can rely on ros2 pkg executables. For instance, you can run as follows. If the command outputs nothing, this means that no nodes were found.
```
ros2 pkg executables python_package
```

After you've finished all the deliverables, launch the two nodes and test out these ROS 2 commands:

ros2 topic list
ros2 topic info drive
ros2 topic echo drive
ros2 node list
ros2 node info talker
ros2 node info relay

## Creating nodes with publishers and subscribers

You can use `ros2 interface show <msg_name>` to see the definition of messages.

* Create two nodes in the package we just created with Python.
* The first node will be named `talker.py` and needs to meet these criteria:
    - `talker` listens to two ROS parameters `v` and `d`.
    - `talker` publishes an `AckermannDriveStamped` message with the `speed` field equal to the `v` parameter and `steering_angle` field equal to the `d` parameter, and to a topic named `drive`.
    - talker publishes as fast as possible.
*The second node will be named `relay.py` and needs to meet these criteria:
    - `relay` subscribes to the drive `topic`.
    - In the subscriber callback, take the speed and steering angle from the incoming message, multiply both by 3, and publish the new values via another `AckermannDriveStamped` message to a topic named `drive_relay`.

Note the following topic names for your publishers and subscribers:

    LaserScan: /scan
    Odometry: /ego_racecar/odom, specifically, the longitudinal velocity of the vehicle can be found in twist.twist.linear.x
    AckermannDriveStamped: /drive


The LaserScan Message

[LaserScan](http://docs.ros.org/en/noetic/api/sensor_msgs/html/msg/LaserScan.html) message contains several fields that will be useful to us. You can see detailed descriptions of what each field contains in the API. The one we'll be using the most is the `ranges` field. This is an array that contains all range measurements from the LiDAR radially ordered. You'll need to subscribe to the `/scan` topic and calculate iTTC with the LaserScan messages.
The Odometry Message

Both the simulator node and the car itself publish [Odometry](http://docs.ros.org/en/noetic/api/nav_msgs/html/msg/Odometry.html) messages. Within its several fields, the message includes the cars position, orientation, and velocity. You'll need to explore this message type in this lab.

The AckermannDriveStamped Message

You've already used [AckermannDriveStamped](http://docs.ros.org/en/jade/api/ackermann_msgs/html/msg/AckermannDriveStamped.html) in the previous lab. It will be the message type that we'll use throughout the course to send driving commands to the simulator and the car. In the simulator, you can stop the car by sending an `AckermannDriveStamped` message with the `speed` field set to 0.0.

To use your new package and executable, first open a new terminal and source your main ROS 2 installation. From inside the ros2_ws directory,
```
source install/setup.bash
```
-->