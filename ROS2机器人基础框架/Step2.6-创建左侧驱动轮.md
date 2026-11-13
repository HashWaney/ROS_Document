**一、建立左驱动轮**
采用双轮差速底盘, 左右轮将使用`continuous`类型的Joint, 使轮子以后可以绕轮轴连续旋转.
![[Pasted image 20261112104004.png]]

本次使用的临时尺寸:

| 参数                  | 示例值     |
| ------------------- | ------- |
| 轮子半径                | 0.05m   |
| 轮子宽度                | 0.04m   |
| 左轮中心X               | 0m      |
| 左轮中心Y               | 0.17m   |
| 左轮中心Z(相对于base_link) | -0.025m |

**二、修改robot.urdf.xacro**
1、 添加轮子尺寸参数
在已有的base_height参数后面添加:
```
<!--驱动轮临时参数-->
<xacro:property  name="wheel_radius" value="0.05"/>
<xacro:property  name="wheel_width" value="0.04"/>
```

2、添加左轮Link和Joint
在文件末尾</robot>之前添加:
```
<!--左侧驱动轮-->
<link name="left_wheel_link">
	<visual>
		<origin xyz="0 0 0" rpy="1.57979632679 0 0 "/>
		<geometry>
			<cylinder radius="${wheel_radius}"
					  length="${wheel_width}"/>
		</geometry>
	</visual>
</link>

<!--左侧驱动轮旋转关节-->
<joint name="left_wheel_joint" type="continuous">
	<parent link="base_link"/>
	<child link="left_wheel_link"/>
	<origin xyz="0 0.17 -0.025" rpy="0 0 0"/>
	<axis xyz="0 1 0"/>
<joint/>
```
这里有两个关键设计:
- continuous: 轮子可以连续转动, 不需要设置旋转角度上下限
- axis xyz="0 1 0": 轮子绕Y轴旋转, 符合当前机器人X轴向前的坐标约定

**三、本次只验证URDF结构**
执行: xacro指令
```
xacro src/robot_descripion/urdf/robot.urdf.xacro \
-o /tmp/robot_foundation.urdf
```

然后检查 `grep -E 'name="left_wheel_(link|joint)"' /tmp/robot_foundation.urdf`, 预期输出:
```
<link name="left_wheel_link">
<joint name="left_wheel_joint" type="continuous">
```



### Step2.6.2: 配置joint_state_publisher

一、 为什么需要joint_state_publisher
目前有两个关节

| Joint                | 类型         | TF发布       |
| -------------------- | ---------- | ---------- |
| base_footprint_joint | fixed      | /tf_static |
| left_wheel_joint     | continuous | /tf        |

`robot_state_publisher` 需要接收到`/joint_states`中的关节角度, 才能计算左轮的动态TF. 因此需要引入`joint_state_publisher`, 为尚未连接真实硬件的关节提供模拟状态, 数据链路是:


![[Pasted image 20261112114309.png]]


二、 开始执行

1、 终端A: 重新启动机器人模型
先停止之前的Launch, 然后执行:
```
source /opt/ros/jazzy/setup.bash
source install/setup.bash

ros2 launch robot_description display.launch.py
```
确保Launch已重新加载包含左轮的URDF, 如果RViz2暂时没有显示左轮, 那是因为动态关节的TF还没有发布.

2、终端B: 启动joint_state_publisher
```
source /opt/ros/jazzy/setup.bash
ros2 run joint_state_publisher joint_state_publisher
```
Jazzy的`joint_state_publisher` 可以自动从`/robot_description`获取模型, 并未可活动关节持续发布状态, 默认频率为10Hz.

3、终端C: 验证关节状态
```
source /opt/ros/jazzy/setup.bash

ros2 topic echo /joint_states --once
```
预期包含:
```
header:
  stamp:
    sec: 1794455353
    nanosec: 273232782
  frame_id: ''
name:
- left_wheel_joint
position:
- 0.0
velocity: []
effort: []
---
```
说明左轮的关节状态已经发布了. 再接着验证TF:
```
ros2 run tf2_ros tf2_echo base_link left_wheel_link
```
预期会输出:
```
 Translation: [0.000, 0.170, -0.025]
- Rotation: in Quaternion (xyzw) [0.000, 0.000, 0.000, 1.000]
- Rotation: in RPY (radian) [0.000, -0.000, 0.000]
- Rotation: in RPY (degree) [0.000, -0.000, 0.000]
- Matrix:
```

其中`Y=0.17m` `Z=-0.025m` 是上一节定义的左轮相对base_link的位置.

![[Pasted image 20261112113909.png]]

### Step 2.6.3 将joint_state_publisher集成到Launch文件

目的是为了让之前手动启动的`joint_state_publisher`随机器人模型一起启动

一、修改display.launch.py

找到现有的`robot_state_publisher` 节点定义, 在它下面新增:
```
# 发布可运动关节的模拟状态
joint_state_publisher = Node(
	package="joint_state_publisher"
	executable="joint_state_publisher",
	name="joint_state_publisher",
	output="screen"
	
)
```
这里不需要再次传入URDF, 因为`joint_state_publisher` 可以订阅`/robot_description`, 自动获取模型中的可运动关节并发布`/joint_states`. 然后找到文件末尾:
```
return LaunchDescription([
	robot_state_publisher,
	# 新增joint节点
	joint_state_publisher,
	rviz
])
```

二、重新构建并启动
先关闭之前手动启动的`joint_state_publisher` 避免两个节点同时发布`/joint_states'. 然后停止旧的Launch, 在终端执行:
