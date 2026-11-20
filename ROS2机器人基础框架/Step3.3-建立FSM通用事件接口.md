目前`robot_system_manager` 已经完成:
- 六状态FSM定义
- INIT->IDLE状态转换
- /system/state 状态发布
- /system/get_state 状态查询
- 非法转换拒绝

本次只实现一个新功能: 通过统一的Service 接收FSM事件, 并返回状态转换结果和拒绝原因

### 一、先确定接口设计
我们不再为每个事件单独创建一个Service, 而是使用统一接口:
```
/system/send_event
```

数据流如下:
![[Pasted image 20261120113811.png]]
由于现有`std_srvs/srv/Trigger` 无法携带事件名称, 我们创建独立的`robot_interfaces` Package存放自定义Service. 


### 二、创建robot_interfaces
先检查依赖:
```
source /opt/ros/jazzy/setup.bash
ros2 pkg prefix rosidl_default_generators
```
如果能够找到, 执行指令:
```
cd ~/robot_foundation_ws

ros2 pkg create robot_interfaces --build-type ament_cmake --license Apache-2.0

mkdir -p robot_interfaces/srv
```
创建`SendEvent.srv`文件
```
vi robot_interfaces/srv/SendEvent.srv
```
填入内容:
```
string event
string source 
---
bool accepted

```
