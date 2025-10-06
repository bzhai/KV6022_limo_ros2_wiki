## Buggy Gazebo
If Gazebo crashes, it may not have fully crashed. It may just be the GUI that crashed. You usually can get
the GUI back by running, in yet another terminal window:
```
ros2 launch gazebo_ros gzclient.launch.py
```