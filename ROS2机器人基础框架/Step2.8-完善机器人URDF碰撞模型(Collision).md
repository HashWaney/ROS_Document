本步目标: 为底盘主体、左右驱动轮和辅助轮万向轮配置碰撞几何, 并通过URDF解析验证.

**一、当前模型与目标**

已有的机器人结构:
```
robot_foundation
└── base_footprint
    └── base_link
        ├── left_wheel_link
        ├── right_wheel_link
        └── caster_mount_link
            └── caster_swivel_link
                └── caster_wheel_link
```

本次需要配置Collision的是四个实体部件:

| Link              | 碰撞几何     | 尺寸              |
| ----------------- | -------- | --------------- |
| base_link         | Box      | 0.40x0.30x0.10m |
| left_wheel_link   | Cylinder | 半径0.05m、宽0.04m  |
| right_wheel_link  | Cylinder | 半径0.05m、宽0.04m  |
| caster_wheel_link | Cylinder | 半径0.03m、宽0.025m |
`base_footprint` 、`caster_mount_link`、`caster_swivel_link` 目前只是坐标或运动学连接节点, 不需要额外添加碰撞几何.

**二、修改Xacro文件**

1、底盘主体Collision
在`base_link`的`</visual>`后添加:
```
<collision>
	<origin xyz="0 0 0.04" rpy="0 0 0"/>
	<geometry>
		<box size="${base_length} ${base_width} ${base_height}"/>
	</geometry>
</collision>
```
2、左右驱动轮Collision
分别找到`left_wheel_link` 和 `right_wheel_link` , 在各自的`</visual>`后添加相同的内容:
```
<collision>
	<origin xyz="0 0 0" rpy="1.57079632679 0 0"/>
	<geometry>
		<cylinder radius="${wheel_radius}"
				  length="${wheel_width}"/>
	</geometry>
</collision>
```
这里沿用Visual中的旋转设置, 让圆柱体轴线与轮轴Y轴一致

3、辅助万向轮Collision
找到`caster_wheel_link` ,在其`</visual>`后添加:
```
<collision>
	<origin xyz="0 0 0" rpy="1.57079632679 0 0"/>
	<geometry>
		<cylinder radius="${caster_radius}"
				  length="${caster_width}"/>
	</geometry>
</collision>
```

**三、统一验证Collision模型**
执行下列指令:
```
cd ~/robot_foundation_ws
xacro src/robot_description/urdf/robot.urdf.xacro -o /tmp/robot_foundation.urf
```
