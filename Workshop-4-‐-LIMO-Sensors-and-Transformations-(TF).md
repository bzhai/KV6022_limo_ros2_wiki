The objectives of this workshop are
* Inspect LIMO robot's sensors and the topics they publish.
* Visualize the TF tree and display the closest laser point as a marker in RViz
* Write a safe-stop behaviour
* Extend it to a simple obstacle-avoidance controller.

### 1. Get to know LIMO sensors
* Pull changes from the [repo](https://github.com/kivrakh/KV6022_limo_ros2) while you are in your root workspace of `KV6022_limo_ros2/`:
```
git pull origin main
```
* Build and Source the workspace
* Open up the simulation environment and find out what type of sensor data they are and which topics they publish.
    * Hint: You should observe and echo
        * `/scan` topic in type of [LaserScan](https://docs.ros.org/en/noetic/api/sensor_msgs/html/msg/LaserScan.html)
```
header:
  stamp:
    sec: 4
    nanosec: 974000000
  frame_id: laser_link
angle_min: -2.0    # rad (~ -114.6°)
angle_max: 2.0     # rad (~ +114.6°)  
angle_increment: 0.011142061091959476    # rad (~0.6384°)
time_increment: 0.0
scan_time: 0.0
range_min: 0.11999999731779099   # m
range_max: 8.0                   # m
ranges:                          # length ~360
- 0.26958101987838745
- 0.25591370463371277
- 0.2722546458244324
- 0.26311302185058594
- 0.2714007496833801
...
```
* `/limo_camera/image` topic in the type of [Image](https://docs.ros.org/en/noetic/api/sensor_msgs/html/msg/Image.html) message
* `/imu` topic in the type of [IMU](https://docs.ros.org/en/noetic/api/sensor_msgs/html/msg/Imu.html) message
* Open also RViz visualisation to explore available sensors (e.g., (depth) camera, LiDAR, IMU) in detail and make the robot move (teleop) and observe the output of the different sensors.
    * Hint: Add the above topics to Rviz either by topic or by adding following displays and setting topics: `LaserScan`, `Image`, `TF`, `RobotModel`. Confirm the `fixed frame` in RViz is set to `odom`.

<img src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/sensors.png" width="800">

* Determine the maximum range at which the proximity sensors on your robot can detect an object. Is there also a minimum range or can objects be detected even if they are placed in direct contact with the sensor?
   * Hint: Write a Python node to subscribe to the `/scan` topic and print the distance to the nearest and furthest obstacle in meters. You can make the robot move using keyboard teleop.

### 2. TF tree and Publishing the closest point as a Marker
* Display the tf tree of the LIMO robot (`ros2 run rqt_tf_tree rqt_tf_tree`) and understand what a frame is (ROS 2 tf2 [theoritical](https://docs.ros.org/en/humble/Concepts/Intermediate/About-Tf2.html) and [practical](https://docs.ros.org/en/humble/Tutorials/Intermediate/Tf2/Introduction-To-Tf2.html) introduction might help, as might this [paper](http://wiki.ros.org/Papers/TePRA2013_Foote?action=AttachFile&do=view&target=TePRA2013_Foote.pdf)) 
    * If `rqt_tf_tree` is not installed. install with 
```
sudo apt update && sudo apt install ros-humble-rqt-tf-tree
```

![LIMO frames](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/frames.png)

* Work out to display the position of the robot's laser (which frame does it have?) in global (`/odom`) coordinates you may either implement Python code following the [TranformListener example](https://docs.ros.org/en/humble/Tutorials/Intermediate/Tf2/Writing-A-Tf2-Listener-Py.html) or the given [tf_listener.py](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/example_codes/tf_listener.py), or figure out how to use a command-line tool: `ros2 run tf2_ros tf_echo`

* TF lets you transform a point measured in one frame to another. Here, we take the closest laser point (in `laser_link`) and transform it into `odom`, then publish a marker in `odom` frame. In order to that you will complete the Python node provided and your code would go in part of the file [closest_lidar_point.py](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/example_codes/closest_lidar_point.py) where it says `START WRITING YOUR CODE HERE`. Some useful pointers:
   * `LaserScan.ranges[i]` or the points in the [LaserScan](http://docs.ros.org/melodic/api/sensor_msgs/html/msg/LaserScan.html) message are stored in polar coordinates in a compressed way to save space. You'll need to loop through the ranges array, calculating the angle for that specific point using the equation:
* `closest_angle = msg.angle_min + (index of closest_range * msg.angle_increment)`

 Next you can convert to `(x,y)` in the scan's frame using:
* `x = closest_range * cos(closest_angle), 
   y = closest_range * sin(closest_angle),
   z= 0`

Then the code store these x, y and z in [PointStamped](https://docs.ros.org/en/noetic/api/geometry_msgs/html/msg/PointStamped.html) message to transform (convert) that point from the scan frame (e.g., `laser_link`) into `odom`, and publish it as [Marker](https://docs.ros.org/en/noetic/api/visualization_msgs/html/msg/Marker.html) message.
* You can visualise the in RViz either by adding a Marker display and subscribing to `/closest_scan_pose` or by adding a topic.
<img src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/nearest_laser.png" width="800">

### 3. Safely stop behavior
* A mobile robot may not stop exactly in front of an obstacle; it leaves some extra space for safety, perhaps 50 cm or 30 cm. Define a threshold or more thresholds (for example, when the object is "close", "middle”, "far"), the minimum safe distance to an object, and program your robot so that it stops at this distance from an object. 
    * Hint: Publish to `/cmd_vel` a zero velocity when `min_front_laser_scan < safety_radius`. You may consider to extend your code you wrote in Task 1.

### 4. Writing a Reactive Obstacle Avoidance Controller
* Instead of stopping against the obstacle encountered, make the robot avoid obstacles accordingly. A simple method is to compare left vs right range distances and turn toward the more open side.
* Slice the scan into more sectors (front-left, front, front-right) and design a simple controller. You should see that the length of the ranges array is 360 laser beams in the ~229-degree range of [-114.5, 114.5]. So if we want to read the LaserScan data on the left, in front and on the right of the robot, we can infer accordingly:
```
 # value at -114.5 degree, this is the right most laser beam
 print msg.ranges[0]
 # value at 90 degree
 print msg.ranges[180]
 # value at 114.5 degree, this is the left most laser beam
 print msg.ranges[360]
```   