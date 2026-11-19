
本次只实现一个功能: **通过模拟自检成功事件, 让机器人从`INIT` 转换为`IDLE`**, 不接入真实传感器、故障管理或运动控制

### 一、设计本次状态轮换

![[Pasted image 20261119164701.png]]

这次重点不只是切换状态, 而是建立一个以后能够复用的状态转换机制.

| 当前状态 | 事件           | 目标状态 | 处理结果   |
| ---- | ------------ | ---- | ------ |
| INIT | INIT_SUCCESS | IDLE | 允许     |
| IDLE | INIT_SUCCESS | IDLE | 拒绝重复转换 |
| 其他状态 | INIT_SUCCESS | 不变   | 拒绝非法转换 |

### 二、修改system_manager_node.py

1、 添加状态转换规则

在`SystemState`枚举定义结束后、`SystemManager`类之前, 添加:
```
# 状态转换规则表
TRANSITIONS = {
	(SystemState.INIT, "INIT_SUCCESS"): SystemState.IDLE,
}
```
这里使用表驱动方式, 后续新增`START_MANUAL` 、`START_AUTO`、`STOP` 等事件时, 可以逐步增加规则, 而不是编写大量潜逃if.

2、创建模拟自检成功Service

在`SystemManager.__init__()`中, 找到原有的:
```
self._get_state_srv = self.create_service(
	Trigger,
	"/system/get_state",
	self._handle_get_state
)
```
在它后面添加:
```
self._mock_init_srv = self.create_service(
	Trigger,
	"/system/mock_init_success",
	self._handle_init_success
)
```
mock 明确表示该接口仅用于研发测试, 不代表真实机器人已经完成自检.

3、 添加状态转换函数

在`SystemManager` 类中, 与`_handle_get_state` 同一级别, 添加:
```
def _try_transition(self,event):
	key =(self._state,event)
	target=TRANSITIONS.get(key)
	if target is None:
		reason = f"Rejected : {self._state.value} + {event}"
		self.get_logger().warn(reason)
		return False, reason
	previous = self._state
	self._state = target
	self.get_logger().info(f"FSM: {previous.value}-> {target.value}")
	self._publish_state()
	return True, self._state.value
	
def _handle_init_success(self,request,response):
	success, message = self._try_transition("INIT_SUCCESS")
	response.success = success
	response.message = message
	return response

```

这样所有状态都必须经过`_try_transition()`检查, 不允许外部请求直接修改`_state`, 原来的`_handle_get_state()` 和`_publish_state()` 保持不变.


### 三、构建并运行

先停止原来正在运行的状态管理节点, 在终端A中执行

```
cd ~/robot_foundation_ws
source /opt/ros/jazzy/setup.bash

colcon build --packages-select robot_system_manager --symlink-install

source install/setup.bash

ros2 run robot_system_manager system_manager_node
```
预计输出:
```
[INFO] [1795080360.981813614] [robot_system_manager]: System state: INIT
```

然后在终端B: 持续订阅状态
```
source install/setup.bash
ros2 topic echo /system/state --qos-reliability reliable --qos-dura
```