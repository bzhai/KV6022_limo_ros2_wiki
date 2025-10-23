In this workshop, you are expected to:
* Inspect the LIMO robot's sensors and the topics they publish.
* Visualize the TF tree and display the position of the closest laser scan reading in RViz
* Write a safe-stop behaviour based on distance to an object.
* Extend the safe-stop behaviour task to a simple obstacle-avoidance controller.

### 1. Get to know LIMO sensors

* Open up the simulation environment and find out what type of sensor data they are and which topics they publish.
    * Hint: You should observe
        * /scan → sensor_msgs/msg/LaserScan
        * /camera/image_raw → sensor_msgs/msg/Image
        * /imu → sensor_msgs/msg/Imu
* Open also Rviz visualisation to explore available sensors (e.g., (depth) camera, LiDAR, IMU) in detail and make the robot move (teleop) and observe the output of the different sensors.
    * Hint: Add the above topics to Rviz either by topic or by adding following displays and setting topics: `LaserScan`, `Image`, `TF`, `RobotModel`. Confirm the `fixed frame` in RViz is set to `odom`.

<img src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/sensors.png" width="800">

* Determine the maximum range at which the proximity sensors on your robot can detect an object. Is there also a minimum range or can objects be detected even if they are placed in direct contact with the sensor? If there is a minimum range, find out why closer objects cannot be detected.
   * Hint: Write a Python node (`laser_scan_subscriber.py`) to subscribe to the `/scan`} topic and print the distance to the nearest and furthest obstacle in meters. You can make the robot move using keyboard teleop.

### 2. TF tree and Determining the range of a distance sensor
* Display the tf tree of the LIMO robot (`ros2 run rqt_tf_tree rqt_tf_tree`) and understand what a frame is (https://docs.ros.org/en/humble/Tutorials/Intermediate/Tf2/Introduction-To-Tf2.html might help, as might this [scientific paper](http://wiki.ros.org/Papers/TePRA2013_Foote?action=AttachFile&do=view&target=TePRA2013_Foote.pdf)) 
    * If `rqt_tf_tree` is not installed. install with `sudo apt update && sudo apt install ros-humble-rqt-tf-tree`

![LIMO frames](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/frames.png)

* Work out to display the position of the robot's laser (which frame does it have?) in global (`/odom`) coordinates you may either implement Python code following the [TranformListener example](https://docs.ros.org/en/humble/Tutorials/Intermediate/Tf2/Writing-A-Tf2-Listener-Py.html) or the given [tf_listener.py](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/example_codes/tf_listener.py), or figure out how to use a command-line tool: `ros2 run tf2_ros tf_echo`

* Modify the Python node that publishes marker `visualization_msgs/Marker` message at the position of the closest laser scan reading and displays it in Rviz. Your code would normally go in part of the file `closest_lidar_point.py` where it says `WRITE YOUR CODE HERE`. Some useful pointers:
   * Determine which frame the laser scans are in!
   * `LaserScan.ranges[i]` or the points in the [LaserScan](http://docs.ros.org/melodic/api/sensor_msgs/html/msg/LaserScan.html) message are stored in polar coordinates in a compressed way to save space. You'll need to loop through the ranges array, calculating the angle for that specific point using the equation:
* `closest_angle = msg.angle_min + (index of closest_range * msg.angle_increment)`

 Next you can convert to `(x,y)` in the scan's frame using:
* `x = closest_range * cos(closest_angle), y = closest_range * sin(closest_angle)`

Then store these x and y in [PointStamped](https://docs.ros.org/en/noetic/api/geometry_msgs/html/msg/PointStamped.html) message to transform (convert) that point from the scan frame (e.g., `laser_link`) into `odom`, and publish it as [Marker](https://docs.ros.org/en/noetic/api/visualization_msgs/html/msg/Marker.html) message.
* You can visualise the in RViz either by adding a Marker display and subscribing to `/closest_scan_pose` or by adding a topic.
![Rviz Image](url)


### 3. Safely stop
* A mobile robot like a self-driving car does not stop exactly in front of an obstacle; it leaves some extra space for safety, perhaps 1 m or 50 cm. Define a threshold or more thresholds (for example, when the object is "close", "middle”, "far"), the minimum safe distance to an object, and program your robot so that it stops at this distance from an object. 
    * Hint: You extend your code you wrote in Task 1.

### 4. Writing an Obstacle Avoidance Controller
* Instead of stopping against the obstacle encountered, make the robot avoid obstacles accordingly. A simple method is to compare left vs right range distances and turn toward the more open side.
* Slice the scan into more sectors (front-left, front, front-right) and design a simple controller.