目前`robot_system_manager` 已经具备完整的六状态FSM、异常优先级和复位规则, 接下来需要解决一个问题: 如何知道某个状态为什么发生改变, 以及哪些事件被FSM拒绝?
本次只实现一个完整功能: 通过`/system/state_event` Topic发布结构化事件记录.

### 一、设计事件记录机制
![[Pasted image 20261123142825.png]]

新增消息`robot_interfaces/msg/StateEvent`, 包含:

| 字段             | 类型     | 说明      |
| -------------- | ------ | ------- |
| stamp          | Time   | 事件处理时间  |
| event          | string | 事件名称    |
| source         | string | 事件来源    |
| previous_state | string | 事件处理前状态 |
| current_state  | string | 事件处理后状态 |
| accepted       | bool   | 是否接受事件  |
| reason         | string | 接受或拒绝原因 |
这里需要区分两个Topic
- `/system/state`:  只关注当前整机状态
- `/system/state_event`:  记录每一次事件处理结果, 包括拒绝事件, 解除故障但状态尚未改变的事件.
### 二、扩展robot_interfaces

1、创建消息文件

```
cd ~/robot_foundation_ws/src/robot_interfaces

mkdir -p msg

vi msg/StateEvent.msg
```

写入内容:
```
builtin_interfaces/Time stamp
string event 
string source
string previous_state
string current_state
bool accepted
string reason
```

2、修改CMakeLists.txt
保留原有`SendEvent.srv`, 在现有配置中增加`StateEvent.msg` 和 时间接口依赖, 相关配置应为:
```
find_package(ament_cmake REQUIRED)
find_package(rosild_default_generators REQUIRED)
find_package(builtin_interfaces REQUIRED)

rosidl_generate_interfaces(${PROJECT_NAME}
	"srv/SendEvent.srv"
	"msg/StateEvent.msg"
	DEPENDENCIES builtin_interfaces
)

ament_export_dependencies(rosidl_default_runtime)

ament_package()
```

注意不要重复添加`ament_package()`, 然后在`robot_interfaces/package.xml` 的依赖区添加:
```
<depend>builtin_interfaces</depend>
```
其他依赖保持不变, 这种消息定义和依赖组织方式与ROS 2 Jazzy接口包的配置一致.

### 三、修改system_manager_node.py

