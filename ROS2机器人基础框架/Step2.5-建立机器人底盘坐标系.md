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

这里使用`fixed` Joint, 代表两个坐标系之间的关系固定. `robot_state_publisher` 会自动将固定关节的TF发布到`/tf_static`