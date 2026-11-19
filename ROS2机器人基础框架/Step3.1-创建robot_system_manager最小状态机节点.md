
本次只完成一个完整的Step: **创建整机状态管理节点, 让它能够启动,维护当前状态,并通过ROS 2 Service查询状态**, 暂时不接入行为树、运动控制、网络通信和故障管理


### 一、本次的最小架构

![[Pasted image 20261119143805.png]]

目前先保留已确认的六个整机状态:
`INIT` 、`IDLE` 、`MANUAL`、`AUTO` 、`FAULT`、`ESTOP`, 本步骤只实现`INIT`初始状态和状态查询.  状态转换逻辑放在下一步; 技术选择采用
`Python + rclpy + ament_python`, 先验证状态机架构. 理由是状态管理属于低频控制逻辑, 首阶段没有硬实时计算要求; 使用Python可以更快验证状态流转和服务接口. 将来需要与C++运动节点通信时, 直接使用标准ROS 2消息即可.

接口约定:

| 接口                  | 类型                     | 作用     |
| ------------------- | ---------------------- | ------ |
| `/system/state`     | `std_msgs/msg/String`  | 发布整机状态 |
| `/system/get_state` | `std_srvs/srv/Trigger` | 查询当前状态 |

### 二、创建ROS 2 Package

在Ubuntu终端执行:
```
cd ~/robot_foundation_ws
source /opt/ros/jazzy/setup.bash

# 核验所需依赖
ros2 pkg prefix rclpy
ros2 pkg prefix std_msgs
ros2 pkg prefix std_srvs
```
三项都能找到后, 创建Package, 执行下列指令:
```
cd ~/robot_foundation_ws/src

ros2 pkg create robot_system_manager \
	--build-type ament_python \
	--license Apache-2.0 \
	--dependencies rclcpy std_msgs std_srvs
```

