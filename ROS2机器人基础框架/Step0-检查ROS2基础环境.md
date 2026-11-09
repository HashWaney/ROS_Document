在Ubuntu24.04虚拟机终端运行:

```
uname -m
lsb_release -ds

source /opt/ros/jazzy/setup.bash
echo $ROS_DISTRO

ros2 pkg list | grep -E \
'^(xacro|robot_state_publisher|joint_state_publisher|rviz2|ros2_control)$'
```

当前需要确认
1、CPU架构是否为aarch64
2、Ubuntu是否为24.04
3、ROS 2 Jazzy能否正常加载
4、URDF开发所需的基础zu