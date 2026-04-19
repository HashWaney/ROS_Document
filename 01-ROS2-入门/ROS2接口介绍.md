
## 什么是接口

接口其实就是一种规范.
```
std_msgs/msg/String
std_msgs/msg/UInt32
```
这两种数据类型分别代表字符串和32位二进制的整型数据, 是ROS2提前定义的一种规范.

## 为什么使用接口
举个例子: 不同的厂家生产出不同的类型的激光雷达, 每种雷达驱动方式、扫描速率等等都不相同.
当机器人进行导航时, 需要激光雷达的扫描数据, 假设没有统一接口, 每次都更换一个种类的雷达, 
都需要重新做程序适配.

在ROS2中定义了一个统一的接口叫做sensor_msgs/msg/LaserScan, 现在几乎每个雷达的厂家都会编写程序将自己雷达的数据变成sensor_msgs/msg/LaserScan格式,提供给用户使用.

## ROS2自带的接口

使用ros2 interface package sensor_msgs 命令可以查看某一个接口包下所有的接口
比如: 传感器类的消息包 sensor_msgs
```
ros2 interface package sensor_msgs
sensor_msgs/msg/JointState #机器人关节数据
sensor_msgs/msg/Temperature #温度数据
sensor_msgs/msg/Imu #加速度传感器
sensor_msgs/msg/Image #图像
sensor_msgs/msg/LaserScan #雷达数据
.......
```

# 接口文件内容


## 可以定义的接口三种类型
ROS2 提供了四种通信方式:
- 话题-Topics
- 服务-Services
- 动作-Action
- 参数-Parameters
除了参数之外, 话题、动作和服务都支持自定义接口, 每一种通信方式所适用的场景各不相同,所定义的接口也被分为话题接口、服务接口、动作接口三种.


## 接口形式

这三种接口定义形式:
话题接口格式: xxx.msg
```
int64 num
```
服务接口格式: xxx.srv
```
int64 a 
int64 b 
---
int64 sum
```
动作接口格式: xxx.action
```
int32 order
----
int32[] sequence
---
int32[] partial_sequence
```

## 接口数据类型

根据引用方式不同可以分为基础类型和包装类型两类
