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
string previous_state
string current_state
string reason
```
其中:

| 字段             | 作用                    |
| -------------- | --------------------- |
| event          | 事件名称, 例如 INIT_SUCCESS |
| source         | 事件来源标识, 例如cli_test    |
| accepted       | 事件是否被接受               |
| previous_state | 转换前状态                 |
| current_state  | 转换后状态                 |
| reason         | 接受或拒绝原因               |
`source` 目前只用于日志追踪, 不具备身份认证能力.

**配置`CmakeLists.txt`**
保留project(robot_interfaces), 确认以下配置存在, 并且ament_package() 位于最后:

```
find_package(ament_camke REQUIRED)
find_package(rosild_default_generators REQUIRED)

rosidl_generate_interfaces(${PROJECT_NAME}
	"srv/SendEvent.srv"
)

ament_export_dependencies(roidl_default_runtime)

ament_package()
```
不要重复添加已有的`find_package(ament_cmake REQUIRED)` 或 `ament_package()`

**配置package.xml**
在`robot_interfaces/package.xml`的依赖区域添加:
```
<build_depend>rosidl_default_generators</build_depend>
<exec_depend>rosild_default_runtime</exec_depend>
<member_of_group>rosild_interface_packages</member_of_group>
```
原有的`ament_cmake` 构建依赖保持不变.

