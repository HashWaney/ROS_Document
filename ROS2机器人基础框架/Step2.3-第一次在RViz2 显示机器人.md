

1、启动robot_state_publisher

打开Ubuntu终端A, 执行:
```
source /opt/ros/jazzy/setup.bash
cd ~/robot_foundation_ws

ros2 run robot_state_publisher robot_state_publisher \
--ros-args \
-p robot_description:="$(xacro src/robot_description/urdf/robot.urdf.xacro)"
```

正常情况下, 日志应该包含类似内容:
```
[INFO] [1794272527.465109031] [robot_state_publisher]: Robot initialized
```
保持终端A运行, 不要关闭
robot_state_publisher 会把机器人模型发布到/robot_description. 当前只有一个base_link, 没有关节, 因此暂时没有其他Link之间的TF变化.

2、验证模型话题

打开终端B:
```
source /opt/ros/jazzy/setup.bash
ros2 topic list | grep robot_description
```
预期输出:
```
/robot_description
```

3、启动RViz2
继续在终端B执行:
```
rviz2
```

在RViz2中,
  1. 将左侧Global Options -> Fixed Frame 设置为base_link
  2. 点击Add, 选择RobotModel
  3. 在RobotModel设置中, 将Description Source 设置为Topic, 并将Description Topic设置为/robot_description
  4. 观察中央三维视图
![[Pasted image 20261110093211.png]]

