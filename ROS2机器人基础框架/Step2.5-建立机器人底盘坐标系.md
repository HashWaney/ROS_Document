目前的ROS 2基础工程已经支持了
-  通过Launch一键启动机器人模型
- 自动加载RViz 2配置
- 正确显示`base_link` 底盘
### 一、创建base_footprint坐标系

1、为什么需要base_footprint?

目前机器人只有`bash_link`, 后续还需要控制机器人移动、计算里程计、接入激光雷达, 因此需要区分两个坐标系:

| 坐标系            | 职责                     |
| -------------- | ---------------------- |
| base_footprint | 机器人底盘在地面上的参考坐标系        |
| base_link      | 机器人主体参考坐标系, 用于连接轮子、传感器 |
这一步完成后, 机器人结构就会变成:
![[Pasted image 20261111153148.png]]

在当前模型中, 底盘高度为0.15m, 暂时将`base_link`原点设置为高出`base_footprint` 0.075m.
**注意: 0.075m是示例模型的临时参数,不是真实机器人的离地高度.**  后续添加左右驱动轮和万向轮时, 需要重新确定底盘离地间隙及轮轴位置.

2、修改Xacro文件
```
vi ~/robot_foundation_ws/src/robot_description/urdf/robot.urdf.xacro
```

在文件末尾`</robot>` 之前添加:
```
<!--地面投影坐标系-->
<link name="base_footprint"/>

<!--base_footprint 到 base_link的固定连接-->
<joint name="base_footprint_joint" type="fixed">
	<parent link="base_footprint"/>
	<child link="base_link"/>
	<origin xyz="0 0 0.075" rpy="0 0 0"/>
</joint>
```

这里使用`fixed` Joint, 代表两个坐标系之间的关系固定. `robot_state_publisher` 会自动将固定关节的TF发布到`/tf_static`, 原来的`base_link` 定义完全保留, 不需要修改

3、仅仅验证Xacro结构
保存上述修改之后, 执行:
```
cd ~/robot_foundation_ws

xacro src/robot_description/urdf/robot.urdf.xacro \
-o /tmp/robot_foundation.urdf

grep -A 5 'joint name="base_footprint_joint"' \
/tmp/robot_foundation.urdf
```
 预期结果是:
 ```
  <joint name="base_footprint_joint" type="fixed">
    <parent link="base_footprint"/>
    <child link="base_link"/>
    <origin rpy="0 0 0" xyz="0 0 0.075"/>
  </joint>
</robot>
 ```

### 二、验证base_footprint -> base_link坐标变换

这一步只验证三件事:
- robot_state_publisher 能够加载更新后的URDF
- /tf_static 正确发布两个坐标系之间的固定变换
- RViz2能够在base_footprint坐标系下显示机器人模型

1、重新构建并启动机器人模型
```
cd ~/robot_foundation_ws

source /opt/ros/jazzy/setup.bash

colcon build \
--packages-select robot_description \
--symlink-install

source install/setup.bash

ros2 launch robot_description display.launch.py
```

2、验证TF静态坐标变换

**2.1:** 检查静态TF发布者
```
ros2 topic info /tf_static -v
```

验收要求:
- Publisher 为 robot_state_publisher
- Publisher count >= 1
- Durability 为`TRANSIENT_LOCAL`

**2.2** 验证实际坐标变换(核心)
执行以下指令, 该指令会直接通过TF2查询两个坐标系之间的变换, 而不是依赖ros2 topic echo 显示原始消息.
```
ros2 run tf2_ros tf2_echo base_footprint base_link
```
预期输出:
```
At time 0.0
- Translation: [0.000, 0.000, 0.075]
- Rotation: in Quaternion (xyzw) [0.000, 0.000, 0.000, 1.000]
- Rotation: in RPY (radian) [0.000, -0.000, 0.000]
- Rotation: in RPY (degree) [0.000, -0.000, 0.000]
- Matrix:
  1.000  0.000  0.000  0.000
  0.000  1.000  0.000  0.000
  0.000  0.000  1.000  0.075
  0.000  0.000  0.000  1.000
```


3、在RViz2中验证
找到已经启动的RViz2窗口:
- 1. 将Global Options -> Fixed Frame 从base_link改为base_footprint
- 2. 确认RobotModel 状态为OK
- 3. 点击Add->TF, 添加TF显示插件

此时应该能过看到如下坐标关系:
```
base_footprint
	|
	| fixed joint 
	| Z = +0.075m
	|
	▶️ base_link
```
![[Pasted image 20261112090142.png]]


### 三、配置并保存TF可视化
将base_footprint设置为RViz2的参考坐标系, 并保存TF显示配置.

1、修改RViz2显示配置
保持`ros2 launch robot_description display.launch.py` 正常运行.
在RViz2 左侧Displays面板中:
 - 找到Global Options -> Fixed Frame, 修改为`base_footprint`
 - 点击Add-> By display type -> TF -> OK, 添加TF显示组件(如果已经存在, 不需要重复添加)
 - 展开TF组件, 确认Show Names已启用, 方便观察坐标系名称
此时应当能看到两个坐标系
```
base_footprint
	|
	----- base_link
			Z = +0.075m
```
可以调整RViz2视角, 观察两个坐标系的位置差异

2 、保存RViz2配置
在RViz2中选择:
**File -> Save Config As ...**  将文件保存到`/home/hash/robot_foundation_ws/src/robot_description/rviz/robot_display.rviz`
覆盖之前的配置文件.

3、验证保存结果
打开终端, 执行:
```
grep -A 2 'Global Options:' \
~/robot_foundation_ws/src/robot_description/rviz/robot_display.rviz
```
预期输出:
```
Global Options:
	Fixed Frame: base_footprint
```
再执行:
```
grep -E 'Class: rviz_default_plugins/(TF|RobotModel)' \
~/robot_foundation_ws/src/robot_description/rviz/robot_display.rviz
```
预期包含:
```
Class: rviz_default_plugins/RobotModel
Class: rviz_default_plugins/TF
```

![[Pasted image 20261112094050.png]]