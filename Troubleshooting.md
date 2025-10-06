## Buggy Gazebo
If Gazebo crashes, it may not have fully crashed. It may just be the GUI that crashed. You usually can get
the GUI back by running, in yet another terminal window:
```
ros2 launch gazebo_ros gzclient.launch.py
```
## Quick fix on ROS 2 Networking
If we wanted to avoid other devices' nodes, we can use the `ROS_LOCALHOST_ONLY` environment variable to limit communication to only nodes on the same device. To keep communications on your own machine only (Local host only variable), set by adding it to your `~/.bashrc` and the daemon (discovery of nodes) must be restarted for changes in the environment to be propagated:
```
echo 'export ROS_LOCALHOST_ONLY=1' >> ~/.bashrc
source ~/.bashrc
ros2 daemon stop
ros2 daemon start
```