## 1.1 创建工程工作空间
在Ubuntu终端执行:
```
mkdir -p ~/robot_foundation_ws/src
cd ~/robot_foundation_ws
pwd
ls -al
```
预期: pwd 输出/home/用户名/robot_foundation_ws, 并且存在src文件夹

## 1.2 创建第一个ROS2 Package

第一个Package 选择robot_description, 专门管理机器人的物理结构和坐标关系
后续负责:
- URDF/Xarco 机器人模型
- 机器人底盘、轮子、传感器安装位置
- Joint与Link定义
- TF坐标关系
- RViz2模型显示配置
设计原则: robot_description只描述机器人结构,不包含业务逻辑、状态机和运动控制代码.
### 一、执行创建命令
在Ubuntu终端执行:
```
cd ~/robot_foundation_ws/src

ros2 pkg create robot_description \
	--build-type ament_cmake \
	--license Apache-2.0
```

### 二、验证创建结果
```
ls -la ~/robot_foundation_ws/src/robot_description
```
预期目录:
```
~/robot_foundation_ws$ tree
.
└── src
    └── robot_description
        ├── CMakeLists.txt
        ├── include
        │   └── robot_description
        ├── LICENSE
        ├── package.xml
        └── src

6 directories, 3 files
```
其中: 
- package.xml: 描述ROS 2 Package的名称、版本和依赖.
- CMakeLists.txt: 负责构建与安装配置
- 