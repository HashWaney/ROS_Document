
在该阶段,能够理解机器人各个模块如何初始化、如何有序启动、如何释放资源, 以及如何与之前建立的整机FSM协作
### 一、Lifecycle在整个机器人架构中的位置

![[Step4-lifecycle.svg]]
三个模块的职责必须要区分开来

| 模块                | 核心职责              |
| ----------------- | ----------------- |
| System Manager    | 管理整机运行状态          |
| Lifecycle Manager | 协调各业务节点的初始化、激活和停用 |
| Lifecycle Node    | 管理自身资源以及业务逻辑的启停   |
例如机器人开机后, 未来可能需要按照底盘驱动、传感器、定位、导航的依赖关系依次启动. Lifecycle Manager 就负责组织这一个过程, 并在必要模块准备完成后通知整机FSM.

### 二、理解ROS 2 原生Lifecycle
ROS 2 已经提供了标准生命周期状态机, 不需要如Step3一样自己编写状态转换表

![[Step4-lifecycle-status.svg]]
实际开发中最重要的是这个五个回调:

| 回调函数              | 应该做什么             |
| ----------------- | ----------------- |
| `on_configure()`  | 加载配置、创建通信接口、初始化资源 |
| `on_activate()`   | 启动业务处理、定时器、数据发布   |
| `on_deactivate()` | 停止业务处理、保留已配置资源    |
| `on_cleanup()`    | 释放配置阶段创建的资源       |
| `on_shutdown()`   | 完成最终资源清理          |
**普通ROS 2 Timer、Subscription并不会因为继承了`LifecycleNode`就自动停止业务处理, 开发过程中仍然需要正确管理他们**

### 三、创建工程

先校验所需依赖
```
source /opt/ros/jazzy/setup.bash
source install/setup.bash

ros2 pkg prefix rclpy
ros2 pkg prefix lifecycle_msgs
ros2 pkg prefix robot_interfaces
```
如果`lifecycle_msgs` 不存在, 可以通过以下指令安装:
```
sudo apt install ros-jazzy-lifecycle-msgs
```

