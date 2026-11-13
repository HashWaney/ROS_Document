目前机器人已经具备完整的左右双驱动轮结构, 并且能够正确发布Joint States 和 TF. 本次只建立辅助万向轮的安装位置;
### Step2.7.1: 创建辅助万向轮安装坐标系

本次只建立辅助万向轮的安装位置, 不急着创建万向轮的几何体和旋转关节.

**一、确定安装位置**
暂时采用后置单万向轮结构
![[Pasted image 20261113152634.png]]
安装点暂定:

| 参数  | 示例值                 |
| --- | ------------------- |
| X   | -0.14 m(底盘后方)       |
| Y   | 0 m(底盘中轴线)          |
| Z   | 0 m(相对base_link 原点) |
这些是临时建模参数, 尚未确定实际万向轮的安装高度

**二、修改robot.urdf.xacro**

在</robot>之前添加:
```
<!--辅助万向轮安装坐标系-->
<link name="caster_mount_link"/>

<!--安装点固定在机器人主体上->
<joint name="caster_mount_joint" type="fixed">
	<parent link="base_link"/>
	<child  link="caster_mount_link"/>
	<origin xyz="-0.14 0 0" rpy="0 0 0"/>
</joint>
```
这里使用`fixed`关节, 是因为我们定义的是**万向轮安装座**, 并不是万向轮本身.
后续会在安装座下面增加:
- 万向轮转向关节(绕Z轴)
- 万向轮滚动关节(绕轮轴)
这样才能正确描述具有转向自由度的辅助万向轮, 而不只是一个球体代替.

**三、验证URDF**
执行指令:
```
cd ~/robot_foundation_ws
xacro src/robot_description/urdf/robot.urdf.xacro -o /tmp/robot_foundation.urdf
grep -E 5 'name="caster_mount_joint"' /tmp/robot_foundation.urdf
```
预期输出:
```
 <joint name="caster_mount_joint" type="fixed">
    <parent link="base_link"/>
    <child link="caster_mount_link"/>
    <origin rpy="0 0 0" xyz="-0.14 0 0"/>
  </joint>
</robot>
```


### Step2.7.2: 创建辅助万向轮的转向关节

**一、本次要实现什么?**
之前已经创建了万向轮安装座
```
base_link
└── caster_mount_link
```
本次增加一个能够绕Z轴旋转的转向结构:
![[Pasted image 20261113162854.png]]

设计如下:

| 参数      | 设置                  |
| ------- | ------------------- |
| 新Link   | caster_swivel_link  |
| 新Joint  | caster_swivel_joint |
| Parent  | caster_mount_link   |
| Child   | caster_swivel_link  |
| Joint类型 | continuous          |
| 旋转轴     | 0 0 1               |
这里选择`continuous`, 因为万向轮需要能够绕竖直轴自由转向, 注意: 它是被动转向关节, 不代表以后必须要为它配置一个驱动电机. URDF目前只描述运动学结构, 并不模拟轮子自动随运动方向转向的物理行为.

**二、修改Xacro文件**

在现有`caster_mount_joint` 后面、`</robot>` 之前添加:
```
<!--万向轮转向结构-->
<link name="caster_swivel_link/>

<!--万向轮绕Z轴自由转向-->
<joint name="caster_swivel_joint" type="continuous">
	<parent link="caster_mount_link"/>
	<child link="caster_swivel_link"/>
	<origin xyz="0 0 0" rpy="0 0 0"/>
	<axis xyz="0 0 1"/>
</joint>
```
这里暂时使用 `origin="0 0 0"` 表示转向轴与安装座坐标系原点重合. 之前添加轮子支架、轮轴时, 再定义轮子的实际偏移距离.

**三、验证Xacro**
执行指令:
```
cd ~/robot_foundation_ws

xacro src/robot_description/urdf/robot.urdf.xacro -o /tmp/robot_foundation.urdf

grep -A 5 'name="caster_swivel_joint' /tmp/robot_foundation.urdf
```
预期输出内容:
```
 <joint name="caster_swivel_joint" type="continuous">
    <parent link="caster_mount_link"/>
    <child link="caster_swivel_link"/>
    <origin rpy="0 0 0" xyz="0 0 0"/>
    <axis xyz="0 0 1"/>
  </joint>

```

### Step2.7.3: 创建辅助万向轮轮体与滚动关节

建立辅助万向轮的轮体及滚动关节、让它既能绕Z轴转向, 也能绕自身轮轴滚动

**一、本次实现目标**
目前机器人辅助万向轮的结构是:
```
base_link
└── caster_mount_link
    └── caster_swivel_link
```

本次扩展为:
```
base_link
└── caster_mount_link
    └── caster_swivel_link
        └── caster_wheel_link   ← 新增
             (caster_wheel_joint)
```
这样辅助万向轮具有两个自由度:
- `caster_swivel_joint`: 绕Z轴转向
- `caster_wheel_joint`:   绕轮轴Y轴滚动

本次临时参数:

| 参数       | 示例值       |
| -------- | --------- |
| 万向轮半径    | 0.03m     |
| 万向轮宽度    | 0.025m    |
| 相对转向轴X偏移 | -0.10m    |
| 相对转向轴Y偏移 | -0.045m   |
| 滚动轴      | 0  1  0   |
| Joint类型  | continous |
这里的`-0.10m` 用于让轮体位于当前底盘后方, 同时形成万向轮的拖距, 尺寸均为临时可视化参数. 由于`base_link` 离地0.075m, 轮心高度为(0.075-0.045) = 0.03m; 正好等于轮子半径, 轮子底部位于地面高度.

**二、修改Xacro**

1、 添加万向轮参数
在现有轮子参数后添加:
```
<!-- f-->
```