### Overview
In this workshop, you will gain practical experience in localizing a robot within a known 2D map and visualising what the localization algorithm is doing. By the end of this session, you should be able to:

* Publish an existing occupancy grid map in ROS 2 and visualize it using RViz.
* Localise your robot within the given map using Adaptive Monte Carlo localization (AMCL).
* Configure and fine-tune key parameters of AMCL (a particle filter-based localisation algorithm.
* Understand practical issues such as poor initial pose and the kidnapped robot problem.

### Preparation 1: Create LIMO localisation ROS 2 Package
The workspace contains the `src` folder, which houses packages source codes. You will create your new package inside this folder.
* Navigate into the source directory of your `KV6022_limo_ros2` workspace:
```
cd ~/KV6022_limo_ros2/src/
```
* Use the `ros2 pkg create` command to generate the package skeleton. You must specify the build type `ament_python` for Python and any dependencies. For a Python Package:
```
# Replace 'my_ros2_package' with your chosen package name (e.g., `limo_localisaton`)
ros2 pkg create --build-type ament_python my_ros2_package --dependencies rclpy
```
* Inside this new package, create launch, maps, and params folders for the associated files that will be stored and used. Your package structure should look similar to:
<pre>
  limo_localisation/
  <mark>|-- launch</mark>
  |-- limo_localisation
  <mark>|-- maps</mark>
  <mark>|-- params</mark>
  |-- resource
  |-- test
  -- package.xml
  -- package.xml
  -- setup.cfg
  -- setup.py
</pre>
* Modify the `setup.py` so that `launch`, `maps`, and `params` folders are installed correctly:
<pre>
from setuptools import find_packages, setup
<mark>import os</mark>
<mark>from glob import glob</mark>

package_name = 'limo_localisation'

setup(
    name=package_name,
    version='0.0.0',
    packages=find_packages(exclude=['test']),
    data_files=[
        ('share/ament_index/resource_index/packages',
            ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
        <mark>(os.path.join('share', package_name, 'launch'), glob(os.path.join('launch', '*launch.[pxy][yma]*'))),</mark>
        <mark>(os.path.join('share', package_name, 'maps'), glob(os.path.join('maps', '*.[yaml|pgm|png]*'))),</mark>
        <mark>(os.path.join('share', package_name, 'params'), glob(os.path.join('params', '*.[yaml|txt|xml]*'))),</mark>  
        
    ],
    install_requires=['setuptools'],
    zip_safe=True,
    maintainer='mmmmm',
    maintainer_email='mmmm@mmm.com',
    description='TODO: Package description',
    license='TODO: License declaration',
    extras_require={
        'test': [
            'pytest',
        ],
    },
    entry_points={
        'console_scripts': [
        ],
    },
)
</pre>
* Return to the root of your workspace (`KV6022_limo_ros2/`) and build the workspace (which now includes this package) using:
```
colcon build
source install/setup.bash
```
### Preparation 2: Occupancy grid map representation
Maps used for ROS are stored as `.pgm` image files, with an accompanying `.yaml` file. Below is an example of the `.yaml` file for the [map.pgm](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/maps/map.pgm) from the [driving area](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/maps/map.png) used in this workshop.
<pre>
image: map.pgm
mode: trinary
resolution: 0.05
origin: [-4.800000, -4.500000, 0.0]
negate: 0
occupied_thresh: 0.65
free_thresh: 0.25
</pre>
`resolution` states how big a pixel is in metres `resolution: 0.05` is equivalent to `5` cm, and where is the origin of the map (top left pixel) with respect to a world coordinate system (`origin: ...`). The values `free_thresh` and `occupied_thresh` define how greyscale values are converted into the three states (`mode: trinary`): free, occupied, and unknown.

### Preparation 3: Publishing a Map with ROS
* The `map_server` node (provided by the `nav2_map_server` package) needs the filepath to the `.yaml` file as a parameter. The `node_map_server` node publishes the 2D occupancy grid on the `/map` topic.
* Download the [map.yaml](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/maps/map.yaml) and [map.pgm](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/maps/map.pgm) and place them under your package's `maps` folder
* Download [limo_map_server.launch.py](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/launch/limo_map_server.launch.py) file and save it to your `launch` folder. 

Inspect and modify it so to load the correct map. In particular:
    * Modify line `14` from pointing at the file `''` to `'maps/map.yaml'`
    * The `node_lifecycle_manager` node handles the lifecycle state transitions of the map server. In ROS2 the `map_server` is a managed node (see documentation [here](https://design.ros2.org/articles/node_lifecycle.html)). This use of the lifecycle node manager becomes very common when using the navigation stack, where a map needs to be published _before_ another node can use it. 
* For Lifecycle manager to work correctly we need to install Navigation2, using the command:
```
sudo apt install ros-humble-navigation2
```
* Build and source workspace again. Next, run the map server:
```
ros2 launch limo_localisation map_server.launch.py
```
* In another terminal, confirm that the `/map` topic is listed `ros2 topic list`. You could echo this to the terminal `ros2 topic echo /map`, but a list of 1, 0, -1 (obstacle, freespace, unknown) is not very helpful! Instead, let us use RViz to see the map. 
* Start RViz2  `rviz2`. 
  * Set the Fixed Frame to `map`. 
  * Add the `/map` topic and adjust the `Durability Policy` for the topic to `Transient Local`. 
  * For the Grid display, set `Cell Size` to `0.05` to match the map resolution (5cm). 

You should see the map similar to:

<p align="center">
<img width="700" height="500" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/rviz_map.png" />
</p>

> [!WARNING]
> **Odometry warning**
>
> AMCL will **not** work unless a correct `/odom` topic and TF transform (`odom → base_link`) are available.
>
> * If you have completed the **Week 7 EKF workshop**, you should:
>   * Launch your `ekf_node` (e.g. from `robot_localization`) so that it:
>     * publishes the `/odom` topic, and  
>     * provides the `odom → base_link` transform.
>   * Alternatively, update [limo_diff.gazebo](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/limo_description/urdf/limo_diff/limo_diff.gazebo) with the configuration you created in Week7 so that the odometry and TF tree are consistent.
>
> * If you have **not** implemented the EKF yet, ignore this warning:

### Task 1: Running AMCL algorithm on the Map
AMCL uses a particle filter to estimate the robot pose (`x, y, yaw`) on a known map. For the algorithm to work properly, it must have access to the map (`map`), odometry data (`/odom`) and laser scans (`/scan`).
With this data, the algorithm of each iteration predicts motion from odometry, compares the appearance of the map with the current laser scans and updates particle weights and resamples (selects only those locations that best match the collected data with the map).
* Inspect the the provided localization launch file [localisation.launch.py](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/launch/limo_localisation.launch.py) to understand how it works. Download and save it to your `launch` folder. It:
   * runs the `map_server` node, which publishes the map from the `.pgm` and `.yaml` file.
   * runs the `node_lifecycle_manager` to ensure that AMCL will not start without the map being published.
   * localise it with  [AMCL](http://wiki.ros.org/amcl) `node_amcl` node along with its associated configuration parameter file [amcl.yaml](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/params/amcl.yaml). Download `amcl.yaml` into your `params` folder. 
1. For visualising the localization particles, install the nav2 rviz plugins:
```
sudo apt install ros-humble-nav2-rviz-plugins
```
2. Run our simulation environment:
```
ros2 launch limo_gazebosim limo_gazebo_diff.launch.py
```
3. In another terminal, start the localisation launch file:
```
ros2 launch limo_localisation limo_localisation.launch.py
```
4. Start `rviz2`. In RViz:
 add the following visualisation: `ParticeCloud`. Don't forget to change:
* Ensure `Fixed Frame` is `map`
* Add the `ParticleCloud` display:
   * Set `Topic` to `/particle_cloud`
   * Set `Min Arrow` Length to `0.1` so the particles are visible
   * Under Topic → `History Policy`, choose `Keep All`
   * Under Topic → `Reliability Policy`, choose `Best Effort`
* Add a `PoseWithCovariance` display by topic to visualise the estimated pose and covariance.
* Add a `scan` by topic to visualise current laser measurements.
5.  Once everything has loaded, you may notice the robot at (`0, 0`) but do not notice the particle cloud. This is because AMCL needs an initial guess, published on the `/initialpose` topic. RViz can publish to this for you using `2D Pose Estimate` button this allows us to put a large green arrow down approximately where the robot is, and set the orientation:
* Click `2D Pose Estimate`, then click and drag on the map:
   * Click: sets the position.
   * Drag direction: sets the orientation (heading).

This initializes the AMCL algorithm. This can be performed multiple times and AMCL will take a new guess each time, just in case the estimates are poor. 

<p align="center">
<img width="691" height="510" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/amcl-run.gif" />
</p>

You should now see similar to the image above:
  * A cluster of blue arrows (particles) represent potential points where the robot may be
  * A purple ellipse under the robot (position uncertainty)
  * A yellow triangle (orientation uncertainty)

As the robot moves, particles should gradually converge.

6. Now, teleoperate the robot by using keyboard `ros2 run teleop_twist_keyboard teleop_twist_keyboard` and note the behaviour of the amcl particles together with localisation quality. While driving:
* Watch the robot in Gazebo (ground truth motion)
* Observe the particle cloud and pose estimate in RViz
You should see:
 * Initially, particles spread out (high uncertainty)
 * As the robot moves and sees more of the environment, the particle cluster:
      * Tightens around the true pose
      * Uncertainty decreases and the estimate converges

If your robot becomes lost, you can re-initialise all particles uniformly over the map:
```
ros2 service call /reinitialize_global_localization std_srvs/srv/Empty
``` 


### Task 2 - Inaccurate Initial Pose

1. In order to better understand the principle of operation of `amcl`, it is good to give an inaccurate position of the robot. To do this, you can use the `2D Pose Estimate` tool in RViz, and then select a point about 1-2m away from the actual position of the robot (~20 grid cells or more - default grid size is 5cm). The algorithm will attempt to improve the position quality after driving a few meters. Remember to observe the actual robot in Gazebo, not only in RViz, because the initial position in RViz is intentionally wrong.

2. Try different "guesses" which should be medium and far from the actual location and see how these affect the quality of estimation and its convergence.

* Drive the robot a few meters around the environment:
    * Watch Gazebo as this is the ground truth.
    * In RViz, the robot will initially appear at the wrong place.
* Observe:
    * Does AMCL recover?
    * How quickly does the particle cluster move towards the correct pose?
    * How does the amount of movement affect convergence?

### Task 3 - The kidnapped robot problem
See what happens if the robot is suddenly moved to a completely different location without the AMCL algorithm knowing.
* In Gazebo, use the object manipulation tools to drag the robot to a different part of the map instantly (do not drive it there).
* In RViz, AMCL should detect inconsistency in observations and re-spread particles.
* Drive the robot around the environment and observe whether the algorithm is able to handle this problem?
* If the automatic recovery is not obvious. You can re-initialise all particles uniformly over the map:
```
ros2 service call /reinitialize_global_localization std_srvs/srv/Empty
``` 
<p align="center">
<img width="691" height="305" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/kidnapped-robot.gif" />
</p>

### Task 4: AMCL configuration and Parameter Tuning

AMCL has a lot of parameters that influence how well it performs (proper operation). Among the most important are:
* `alphaX` (`alpha1`–`alpha5`) – noise parameters related to odometry accuracy
* `base_frame_id` - robot base frame, here: `base_link`
* `global_frame_id` - global map frame, here: `map`
* `scan_topic` - laser scan input topic, e.g. `/scan`
* `max_particles` / `min_particles` - min/max number of particles (e.g. `50–500`) 
* `laser_max_range` - maximum usable range of the laser (e.g. `12.0`)
* `update_min_d` / `update_min_a` – how far the robot must move (distance/angle) before updating

* These parameters are typically stored in a YAML file, e.g. [amcl.yaml](https://github.com/kivrakh/KV6022_limo_ros2/blob/main/src/example_codes/params/amcl.yaml):
<pre>
amcl:
  ros__parameters:
    use_sim_time: True
    <mark>alpha1: 0.2</mark>
    <mark>alpha2: 0.2</mark>
    <mark>alpha3: 0.2</mark>
    <mark>alpha4: 0.2</mark>
    <mark>alpha5: 0.2</mark>
    <mark>base_frame_id: "base_link"</mark>
    beam_skip_distance: 0.5
    beam_skip_error_threshold: 0.9
    beam_skip_threshold: 0.3
    do_beamskip: false
    <mark>global_frame_id: "map"</mark>
    lambda_short: 0.1
    laser_likelihood_max_dist: 2.0
    <mark>laser_max_range: 12.0</mark>
    laser_min_range: -1.0
    laser_model_type: "likelihood_field"
    max_beams: 60
    <mark>max_particles: 500</mark>
    <mark>min_particles: 50</mark>
    <mark>odom_frame_id: "odom"</mark>
    pf_err: 0.05
    pf_z: 0.99
    recovery_alpha_fast: 0.0
    recovery_alpha_slow: 0.0
    resample_interval: 1
    <mark>robot_model_type: "nav2_amcl::DifferentialMotionModel"</mark>
    save_pose_rate: 0.5
    sigma_hit: 0.2
    tf_broadcast: true
    transform_tolerance: 1.0
    update_min_a: 0.3
    update_min_d: 0.2
    z_hit: 0.5
    z_max: 0.05
    z_rand: 0.5
    z_short: 0.05
    <mark>scan_topic: scan</mark>
    <mark>map_topic: map</mark>
    set_initial_pose: true
    always_reset_initial_pose: true
    first_map_only: false
    initial_pose:
      x: 0.0
      y: 0.0
      z: 0.0
      yaw: 0.0
</pre>

### 4.1. Experiments with parameters
> [!NOTE]
> In your `limo_localisatin` package, these parameters should be provided in `params/amcl.yaml`. Any changes you make there will apply the next time you launch the localisation.
Experiment with [parameters](https://docs.nav2.org/configuration/packages/configuring-amcl.html) of the `amcl` node:
1. Number of particles 
      * Reduce `max_particles` (e.g. to `200`) 
      * Increase `max_particles` (e.g. to `1000`) and observe: Does it converge or how fast does it converge? How does CPU usage look like?, 
2. Resample interval
      * Change `resample_interval` (e.g. `1`, `2`, `5`) and observe: Does AMCL respond more quickly or more slowly to motion?
3. Odometry noise (alpha values)
      * Increase `alpha1–alpha4` to simulate worse odometry. Decrease them to simulate very good odometry. Observe that Does the filter become over-confident or under-confident?

### 4.2. Live tuning with `rqt_reconfigure`

* You can interactively inspect and change all parameter values of the running nodes. Start the reconfigure GUI:
```
ros2 run rqt_reconfigure rqt_reconfigure
```
or alternatively typing:
```
rqt
``` 
Then from the menu `Plugins → Configuration → Dynamic Reconfigure`. 

* Select the `amcl` node from the list on the left. You should be able to visualise all the parameters you can change. 
* Adjust parameters (e.g. particle numbers, alpha values) and observe:
  * The particle cloud in RViz
  * The convergence and accuracy of the pose estimate
This kind of systematic experimentation is exactly what you'll do in real robotic systems when tuning localisation.
<p align="center">
<img width="440" height="330" alt="image" src="https://github.com/kivrakh/KV6022_limo_ros2/blob/main/wiki_images/amcl_reconfigure.png" />
</p>
