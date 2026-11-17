现在机器人已经具备Visual、Collision和完整的运动学关节结构, 接下来给机器人添加质量(Mass)、质心(Center of Mass)和惯性矩阵(Inertial Tensor), 为后续运动动力学与Gazebo仿真打基础.

一、为什么需要Inertial?
在URDF中, 三个属性的作用不同:

| 属性        | 作用         | 状态   |
| --------- | ---------- | ---- |
| Visual    | 机器人外观      | 已完成  |
| Collision | 碰撞几何       | 已完成  |
| Inertial  | 质量、质心、转动惯量 | 本次配置 |

本次使用的示例质量:

| 部件       | 质量    |
| -------- | ----- |
| 底盘主体     | 8.0kg |
| 左驱动轮     | 0.5kg |
| 右驱动轮     | 0.5kg |
| 辅助万向轮轮体  | 0.2kg |
| 当前建模质量合计 | 9.2kg |

本次按照均匀密度的长方体和实心圆柱体估算惯性矩阵. 将来替换成真实硬件时, 优先通过CAD或实际测量得到质量、质心和惯性参数.

二、修改robot.urdf.xacro

1、定义质量参数
在已有的Xacro参数区添加:
```
<!-- 临时质量参数, 单位kg-->
<xacro:property name="base_mass" value="8.0"/>
<xacro:property name="wheel_mass" value="0.5"/>
<xacro:property name="caster_mass" value="0.2"/>
```

这样以后只需要调整质量参数, 惯性矩阵就会自动重新计算

2、给底盘base_link添加Inertial
在`base_link`的`</collision>`后、`</link>` 前添加
```
<!--底盘质量及惯性参数-->
<inertial>
	<origin xyz="0 0 0.04" rpy="0 0 0"/>
	<mass value="${base_mass}"/>
	<inertia
		ixx="${base_mess*(base_width**2+base_height**2)/12.0}"
		ixy="0"
		ixz="0"
		iyy="${base_mass*(base_length**2+base_height**2)/12.0}"
		iyz="0"
		izz="${base_mass*(base_length**2+base_width**2)/12.0}"/>
</inertial>
```

这里采用长方体惯性公司:

		Ixx = m(w^2+h^2)/12
		Iyy = m(l^2+h^2)/12
		Izz = m(l^2+w^2)/12
3、为左右驱动轮添加Inertial
分别找到`left_wheel_link` 和 `right_wheel_link`, 在各自的`</collision>`后添加相同的代码:
```
<!-- 驱动轮质量及惯性参数-->
    <inertial>
            <origin xyz="0 0 0" rpy="0 0 0"/>
            <mass value="${wheel_mass}"/>
            <inertia 
                ixx="${wheel_mass * (3 * wheel_radius**2 + wheel_width**2) / 12.0}"
                ixy="0"
                ixz="0"
                iyy="${wheel_mass*wheel_radius**2/2.0}"
                iyz="0"
                izz="${wheel_mass*(3*wheel_radius**2+wheel_width**2)/12.0}"/>
        </inertial>
```

注意: 圆柱轮子的Visual已经旋转到Y轴方向, 因此惯性矩阵采用Y轴作为滚动轴, 与左右驱动轮的Joint axis-一致

4、为辅助万向轮添加Inertial
找到`caster_wheel_link`, 同样在`</collision>` 后添加:
```
 <!-- 万向轮轮体质量及惯性参数 -->
        <inertial>
            <origin xyz="0 0 0" rpy="0 0 0"/>
            <mass value="${caster_mass}"/>
            <inertia
                ixx="${caster_mass*(3*caster_radius**2+caster_width**2)/12.0}"
                ixy="0"
                ixz="0"
                iyy="${caster_mass*caster_radius**2/2.0}"
                iyz="0"
                izz="${caster_mass*(3*caster_radius**2+caster_width**2)/12.0}"/>
        </inertial>
```
这样四个实体部件都有了各自的质量和惯性定义

三、验证Xacro和惯性参数
执行指令:
```
xacro src/robot_description/urdf/robot.urdf.xacro -o /tmp/robot_foundation.urdf
```
