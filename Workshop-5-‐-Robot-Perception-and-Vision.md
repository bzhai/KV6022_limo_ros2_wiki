### Overview
This workshop explores robot perception and vision using ROS 2 and Gazebo. You will learn to use camera sensors to analyze images from ROS topics, process them with OpenCV and try out object detection using both feature-based and deep-learning method.
By the end of this workshop, you will be able to:
* Subscribe to a camera topic and retrieve images.
* Use OpenCV to display and process images (e.g., filtering, edge detection, color thresholding, contours).
* Experiment with object detection using feature-based and deep learning-based methods.

[comment]: <> (optionally command the robot based on your image processing output)

### Preparations
* Pull changes from the [repo](https://github.com/kivrakh/KV6022_limo_ros2) as new files were added while you are in your root workspace of `KV6022_limo_ros2/`:
```
cd ~/KV6022_limo_ros2/
git pull origin main
```
* Rebuild the updated packages `colcon build --symlink-install`, and source the workspace `source install/setup.bash`.
* Have the LIMO gazebo simulation ready for tasks `ros2 launch limo_gazebosim limo_gazebo_diff.launch.py`.

### Task 1: Explore with rqt tools (visualising image topics)

[Rqt tools](https://docs.ros.org/en/humble/Concepts/Intermediate/About-RQt.html) are very convenient for inspecting image topics. First, install the `rqt_image_view` package by issuing `sudo apt-get install ros-humble-rqt-image-view` and execute the following command to visualise the colour image where image topic is provided as ros parameters/args:
```
ros2 run rqt_image_view rqt_image_view --ros-args -r image:=/limo_camera/image
```
You can also skip the image topic arguments `ros2 run rqt_image_view rqt_image_view` and select an image topic from the list available through GUI.

<img width="600" height="400" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/rqt_image.png" />

* Measure the frequency of an image topic with `ros2 topic hz <topic_name>` to see how much time image topic is published in a second.

### Task 2: Using OpenCV with ROS 2 (CvBridge)

All the source code shown in the demonstration is available [here](https://github.com/kivrakh/KV6022_limo_ros2/tree/main/src/example_codes/example_codes). In particular, have a look at

* [opencv_intro.py](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/example_codes/opencv_intro.py) which shows you how to load an image in OpenCV without ROS and some processing methods. Also look at the official [OpenCV Python tutorials](https://opencv24-python-tutorials.readthedocs.io/en/latest/py_tutorials/py_tutorials.html) to gain a better understanding. [Image processing tutorials](https://docs.opencv.org/4.x/d7/da8/tutorial_table_of_content_imgproc.html), [colour slicing](https://docs.opencv.org/4.x/da/d97/tutorial_threshold_inRange.html), [finding contours](https://docs.opencv.org/4.x/df/d0d/tutorial_find_contours.html) etc.
* [opencv_bridge.py](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/example_codes/opencv_bridge.py) showing you how to use [CvBridge](http://wiki.ros.org/cv_bridge/Tutorials/ConvertingBetweenROSImagesAndOpenCVImagesPython) (this website is for ROS1 but the OpenCV parts are the same for ROS2) to read image from a ROS topic.
* [colour_contours.py](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/example_codes/colour_contours.py) to get an idea about colour slicing (thresholding). Also read about [Changing Colour Spaces](https://opencv24-python-tutorials.readthedocs.io/en/latest/py_tutorials/py_imgproc/py_colorspaces/py_colorspaces.html).

1. Take the example code fragment [opencv_bridge.py](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/example_codes/opencv_bridge.py) and modify it so you can read from the camera of LIMO robot.

<img width="600" height="200" class="center" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/opencv_bridge_ex.png" />

> [!TIP]
> * CvBridge: ROS–OpenCV converter between sensor_msgs/Image and numpy arrays.
> * HSV: Color space that makes hue‑based thresholding more robust than raw RGB.
> * Contours: Curves joining continuous points along a boundary; useful for blob detection.

### Task 3: Colour detection
* [colour_contours_detector.py](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/example_codes/colour_contours_detector.py) node demonstrates how to subscribe to LIMO's image topics, perform colour thresholding and object detection. To test the node, run the simulator and try to place the greenery in front of the robot by adjusting the range of the HSV colour filter accordingly. You might move the robot to detect other green objects around. You should see the debug windows visualising the image processing pipeline. The node outputs the detected objects as the `/object_polygon` topic.

<img width="800" height="300" class="center" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/color_detector.png" />

### Task 5: Object Detection : Feature-based Approach

* Review the documentation for [Find Object 2D](http://introlab.github.io/find-object/) and refer to [find\_object\_2d](http://wiki.ros.org/find_object_2d) wiki page (this is documentation for ROS1 but still applies). Also refer to [here](https://github.com/ros2-gbp/find_object_2d-release) for the usage on ROS2.
* Read about the feature detectors and descriptors used in the [OpenCV Documentation](https://docs.opencv.org/3.0-beta/doc/py\_tutorials/py\_feature2d/py\_table\_of\_contents\_feature2d/py\_table\_of\_contents\_feature2d.html)
* Install the package:
```
sudo apt-get install ros-humble-find-object-2d
```
* Run the object detection node: `ros2 run find_object_2d find_object_2d image:=<image_topic>`
* Put some nice coloured objects in front of the robot (in Gazebo objects can be added in the `insert` tab and under the `models.gazebosim.org` list at the bottom, this may a few minutes to load all the objects). 
* Train and detect objects by marking areas in images, such as a coke can, and inspect object information published on the `objects` topic. To train an object detector, first choose a detector/descriptor via GUI.
    * In the Find‑Object window:
    * `View` → `Parameters`.
    * Under `Feature2D` → `Detector/Descriptor`, start with ORB. You can later try FAST/SIFT or BRISK and compare.
   
<img width="800" height="300" class="center" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/find_object.png" />

*  Then click `Edit` → `Add object from scene`. Drag a rectangle tightly around the object to indicate the object's binding box and confirm (Enter/OK). This creates Object #N in the database.
* (Optional) `File` → `Save objects to a directory` (so you can reuse later in headless mode). 

`ros2 run find_object_2d find_object_2d --ros-args -r image:=/limo_camera/image -p objects_path:=[path_to_your_objects] -p gui:=false`
* You should immediately see green boxes and keypoints when the object is recognized in subsequent frames.
* Inspect detections in the `objects` topic `ros2 topic echo /objects` (see [find_object_2d](http://wiki.ros.org/find_object_2d)) and see if you can find the size and location of the object. You will see lines like `id` `x` `y` `w` `h` `...` (and in /objectsStamped, the same info with header/time). These give you the approximate location and size in image pixels. 

<img width="800" height="300" class="center" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/detected_object.png" />

* To test detector robustness on scale and viewpoint changes, move the robot closer/farther or move the object to observe how detector reflects size changes. If detection drops, add another training view by repeating `Add object from scene` from a new viewpoint.

### Task 6: Object Detection : Deep Learning-based Approach
For CNN-based object detection, we will use ROS 2 [package](https://github.com/mgonzs13/yolo_ros?tab=readme-ov-file) for YOLO models from [Ultralytics](https://github.com/ultralytics/ultralytics) to perform object detection and tracking which first introduced on its [original paper](https://pjreddie.com/media/files/papers/yolo_1.pdf).
* Clone the `yolo_ros` repository into your workspace:
```
cd ~/KV6022_limo_ros2/src
git clone https://github.com/mgonzs13/yolo_ros.git
pip3 install -r yolo_ros/requirements.txt
```
* Install dependencies:
```
cd ~/ros2_ws
rosdep install --from-paths src --ignore-src -r -y
```
* Configure the correct image topic in `./yolo_ros/yolo_bringup/launch/yolo.launch.py` to use the correct image topic and device to use, e.g.:
<pre>
...
device = LaunchConfiguration("device")
device_cmd = DeclareLaunchArgument(
    "device",
    #default_value="cuda:0",
    <mark>default_value="cpu",</mark>
    description="Device to use (GPU/CPU)",
)
....
input_depth_topic = LaunchConfiguration("input_depth_topic")
input_depth_topic_cmd = DeclareLaunchArgument(
    "input_depth_topic",
    <mark>default_value="/limo_camera/depth/image_raw",</mark>
    description="Name of the input depth topic",
)
...
input_depth_topic = LaunchConfiguration("input_depth_topic")
input_depth_topic_cmd = DeclareLaunchArgument(
    "input_depth_topic",
    <mark>default_value="/limo_camera/depth/image_raw",</mark>
    description="Name of the input depth topic",
)
</pre>
* Build and source the package: 
```
cd ~/KV6022_limo_ros2/
colcon build --symlink-install
source install/setup.bash
```
Launch YOLO node:
```
ros2 launch yolo_bringup yolo.launch.py
```
<img width="400" height="400" class="center" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/yolo_result.png" />

Refer to the [yolo_ros](https://github.com/mgonzs13/yolo_ros) repository for further details on published object information (`ros2 topic echo /yolo/detections`).