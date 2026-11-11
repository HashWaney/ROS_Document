
### 2.1 确定机器人模型的坐标系
先以一个通用的移动机器人作为开发模型,未来再根据真实底盘修改尺寸和结构.

1、设计TF坐标树
 ![[Pasted image 20261109171352.png]]
 坐标约定遵循ROS REP-103 和 REP-105:
 - X轴: 机器人前方
 - Y轴: 机器人左方
 - Z轴: 机器人上方
map -> odom 以后由定位或SLAM算法发布; odom->base_footprint由里程计系统发布. 这两段动态关系不写成URDF的固定关节.
对于第一版URDF, 只负责从base_footprint开始描述机器人本体、轮子和雷达的结构

2、本次执行: 创建URDF目录
在Ubuntu中执行:
```
cd ~/robot_foundation_ws/src/robot_description

mkdir -p urdf

ls -al urdf
```
暂时不创建XML文件, 因为还需要确定底盘模型

3、确定第一个机器人模型
考虑到后续要加入运动节点,故障节点和2D雷达, 采用双轮差速移动底盘作为基础模型,尺寸全部保持待定.


### 2.2 创建机器人的第一个Link -base_link

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


### Step2.3: 第一次在RViz2 显示机器人

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


### Step 2.4 :将URDF正式纳入ROS 2 Package, 并实现一键启动

#### 一、配置URDF资源安装

1、为什么需要这一步?

目前工程结构是:
```
robot_foundation_ws/
└── src/
    └── robot_description/
        ├── CMakeLists.txt
        ├── package.xml
        └── urdf/
            └── robot.urdf.xacro
```

虽然robot_description 已经能够成功构建, 但ROS 2 Package不一定会自动安装新建的urdf目录, 将来能够通过 get_package_share_directory("robot_description") 找到模型资源,而不是在代码中硬编码/home/user/...路径.

2、修改CMakeLists.txt
编写该文件, 找到文件末尾的
ament_package()
在其上方添加:
```
# 安装URDF / Xarco 模型资源
install(
	DIRECTORY urdf
	DESTINATION share/$(PROJECT_NAME)
)
```
保留原有内容, 不需要重写整个CMakeLists.txt

3、重新构建Package
执行
```
cd ~/robot_foundation_ws
source /opt/ros/jazzy/setup.bash
colcon build \
	--packages-select robot_description \
	--symlink-install
```
构建完成之后:
```
source install/setup.bash
```

4、验证安装结果
执行:
```
ros2 pkg prefix robot_description
```
然后:
```
ls  "$(ros2 pkg prefix robot_description)/share/robot_description/urdf/"
```
预期结果:
```
robot.urdf.xacro
```
这就说明URDF已经成功作为ROS 2 Package资源安装

#### 二、创建第一个ROS 2 Launch文件

通过一条命令同时启动robot_state_publisher和RViz2, 并自动加载URDF模型, 不添加新的机器人结构,也不涉及运动控制.

1、创建Launch文件
在Ubuntu终端执行:
```
cd ~/robot_foundation_ws/src/robot_description
mkdir -p launch
vi launch/display.launch.py
```

填写以下代码:

```
import os

from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch.substitutions import Command
from launch_ros.actions import Node
from launch_ros.parameter_descriptions import ParameterValue


def generate_launch_description():

    # 1. 获取 robot_description 的安装目录
    package_path = get_package_share_directory("robot_description")

    # 2. 定位 Xacro 模型
    xacro_file = os.path.join(
        package_path,
        "urdf",
        "robot.urdf.xacro"
    )

    # 3. 将 Xacro 转换为 URDF
    robot_description = ParameterValue(
        Command(["xacro ", xacro_file]),
        value_type=str
    )

    # 4. 发布机器人模型
    robot_state_publisher = Node(
        package="robot_state_publisher",
        executable="robot_state_publisher",
        name="robot_state_publisher",
        output="screen",
        parameters=[{
            "robot_description": robot_description
        }]
    )

    # 5. 启动 RViz2
    rviz = Node(
        package="rviz2",
        executable="rviz2",
        name="rviz2",
        output="screen"
    )

    return LaunchDescription([
        robot_state_publisher,
        rviz
    ])

```
`get_package_share_directory`:  获取已安装Package的路径
`Command` : 执行Xacro转换
`ParameterValue`: 将生成的URDF作为字符串参数传入
`robot_state_publisher`: 加载并发布机器人模型
`rviz2`: 启动模型可视化工具
`LaunchDescription`: 统一管理启动过程

以后添加新的机器人节点,也可以采用类似的Launch管理机制;

2、修改CMakeLists.txt
```
install(
	DIRECTORY urdf launch
	DESTINATION share/${PROJECT_NAME}
)
```
这样构建后, Launch文件也会被安装到Package的共享资源目录

3、重新构建并运行
先关闭之前手动启动的robot_state_publisher 和RViz2, 避免出现重复节点, 执行以下指令:

```
cd ~/robot_foundation_ws

source /opt/ros/jazzy/setup.bash
colcon build \
--packages-select robot_description \
--symlink-install

source install/setup.bash
```

构建成功之后运行:
```
ros2 launch robot_description display.launch.py 
```

预期输出日志如下:
```
ros2 launch robot_description display.launch.py 
[INFO] [launch]: All log files can be found below /home/hash/.ros/log/2026-11-10-16-45-15-727522-ros-236511
[INFO] [launch]: Default logging verbosity is set to INFO
[INFO] [robot_state_publisher-1]: process started with pid [236522]
[INFO] [rviz2-2]: process started with pid [236523]
[robot_state_publisher-1] [INFO] [1794300316.173352504] [robot_state_publisher]: Robot initialized
[rviz2-2] [INFO] [1794300317.042928364] [rviz2]: Stereo is NOT SUPPORTED
[rviz2-2] [INFO] [1794300317.043099448] [rviz2]: OpenGl version: 4.5 (GLSL 4.5)
[rviz2-2] [INFO] [1794300317.203246040] [rviz2]: Stereo is NOT SUPPORTED
```

### 三、保存RViz2 配置

本次只做一件事: 将RViz2当前的显示配置保存为.rviz文件, 方便以后自动恢复

1、在RViz中确认显示设置

| 配置项                              | 设置值                |
| -------------------------------- | ------------------ |
| Global Options -> Fixed Frame    | base_link          |
| Displays -> Grid                 | Enabled            |
| Displays -> RobotModel           | Enabled            |
| RobotModel -> Description Source | Topic              |
| RobotModel -> Description Topic  | /robot_description |
| RobotModel -> Visual Enabled     | true               |
确保中央能够正常显示之前创建的灰蓝色长方体, 目前URDF只有base_link, 因此可能存在No tf data 提示, 暂时不需要处理. 后面添加其他Link后, 再检查完整TF树.

2、创建RViz配置目录
打开另一个终端, 执行
```
mkdir -p ~/robot_foundation_ws/src/robot_description/rviz
```

3、从RViz2 保存配置
在RViz2窗口中:
	1. 点击菜单File -> Save Config As ...
	2. 进入目录:  /home/hash/robot_foundation_ws/src/robot_description/rviz/
	3. 文件名填写: robot_display.rviz
	4. 点击保存
注意: 不要保存到install/目录. 后续只维护src/ 中的原始配置文件

4、验证文件是否保存成功
在终端执行:
```
ls -lh ~/robot_foundation_ws/src/robot_description/rviz
grep -E 'Fixed Frame:|RobotModel:|robot_description' \
~/robot_foundation_ws/src/robot_description/rviz/robot_display.rviz
```
预期能看到robot_display.rviz文件, 以及类似配置:


### 四、实现RViz2 自动加载配置

1、修改 display.launch.py

找到之前创建的RViz2节点:
```
rviz = Node(
	package="rviz2",
	executable="rviz2",
	name="rviz2",
	output="screen"
)
```
将其替换为:
```
# RViz2 配置文件路径
rviz_config = os.path.join(
	package_path,
	"rviz",
	"robot_display.rviz"
)

# 启动RViz2 并加载配置
rviz = Node(
	package=“
)

```