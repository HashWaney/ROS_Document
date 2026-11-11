



### Step 2.2 创建机器人的第一个Link -base_link

本次只完成一件事: 使用Xacro定义机器人底盘主体base_link, 并验证能否正确生成URDF

1、理解base_link
base_link 是机器人的本体参考坐标系. 后续左右驱动轮、辅助万向轮和激光雷达都将以它为参考建立连接关系.
![[Pasted image 20261109174654.png]]

先赢一个长方体表示底盘、以下尺寸仅用于开发验证, 并非真实机器人尺寸
|参数|示例值|
|---|---|
|长度 X|0.40 m|
|宽度 Y|0.30 m|
|高度 Z|0.15 m|

2、创建Xacro文件
在Ubuntu终端执行:
```
cd ~/robot_foundation_ws/src/robot_description

vi urdf/robot.urdf.xacro
```

写入以下内容:
```
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro"
       name="robot_foundation">
    <!--开发阶段的底盘尺寸，单位：米-->
    <xacro:property name="base_length" value="0.40"/>
    <xacro:property name="base_width" value="0.30"/>
    <xacro:property name="base_height" value="0.15"/>
    <material name="body_gray">
                <color rgb="0.45 0.50 0.55 1.0"/>
            </material>
    <link name="base_link">
        <visual>
            <origin xyz="0 0 0" rpy="0 0 0"/>
            <geometry>
                <box size="${base_length} ${base_width} ${base_height}"/>
            </geometry>
            <material name="body_gray"/>
        </visual>
    </link>
</robot>
```
目前只定义可视化几何体. 质量、惯量、碰撞体和真实安装位置将在后续阶段补充, 避免把占位参数当作真实硬件规格.

3、验证Xacro文件
保存后执行:
```
cd ~/robot_foundation_ws
source /opt/ros/jazzy/setup.bash
xacro src/robot_description/urdf/robot.urdf.xarco \
-o /tmp/robot.foundation.urdf
grep -A 3 '<box' /tmp/robot_foundation.urff
```
预期输出包含:
```
<box size="0.4 0.3 0.15"/>
```
说明Xacro参数已经成功解析为标准URDF

