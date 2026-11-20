
本次只实现一个功能: 允许机器人在IDLE、MANUAL、AUTO三种状态之间安全切换


### 一、设计状态转换规则

![[Pasted image 20261120152613.png]]

| 当前状态   | 事件            | 目标状态   | 是否允许 |
| ------ | ------------- | ------ | ---- |
| IDLE   | START_MANUAL  | MANUAL | 是    |
| MANUAL | STOP          | IDLE   | 是    |
| IDLE   | START_AUTO    | AUTO   | 是    |
| AUTO   | STOP          | IDLE   | 是    |
| AUTO   | TASK_FINISHED | IDLE   | 是    |
| MANUAL | START_AUTO    | 不变     | 否    |
| AUTO   | START_MANUAL  | 不变     | 否    |

这里有一条重要原则: 手动模式和自动模式不能直接相互切换, 必须先经过IDLE, 目前只是状态逻辑验证, 状态进入到MANUAL或AUTO 不会真正驱动机器人运动

### 二、 修改FSM状态转换表

修改`robot_system_manager/system_manager_node.py` 找到定义的`TRANSITIONS = { (SystemState.INIT, "INIT_SUCCESS"):SystemState.IDLE,}`, 将整个`TRANSITIONS` 替换为:
```
# 机器人整机状态转换规则V1
TRANSITIONS = {

	# 初始化
	(SystemState.INIT, "INIT_SUCCESS"):SystemState.IDLE,
	
	# 手动模式
	(SystemState.IDLE, "START_MANUAL"):SystemState.MANUAL,
	(SystemState.MANUAL,"STOP"):SystemState.IDLE,
	
	# 自动模式
	(SystemState.IDLE, "START_AUTO"):SystemState.AUTO,
	(SystemState.AUTO, "STOP"):SystemState.IDLE,
	(SystemState.AUTO,"TASK_FINISHED"):SystemState.IDLE,
}
```

### 三、重新构建与启动
停止旧的`robot_system_manager`, 执行以下指令:
```
cd ~/robot_foundation_ws

source /opt/ros/jazzy/setup.bash

colcon build --packages-select robot_system_manager --symlink-install

source install/setup.bash

ros2 run robot_system_manager system_manager_node
```
预期启动日志:
```
System state: INIT
```
保持节点运行

### 四、运行状态转换测试

打开第二个终端:
```
source /opt/ros/jazzy/setup.bash
source install/setup.bash

```
1、初始化完成测试
```
ros2 service call /system/send_eve
```