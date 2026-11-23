本次只完成一个功能: 为整机FSM增加故障、急停和受控复位机制, 继续使用现有的`robot_system_manager`, 不创建新Package, 不修改`SendEvent.srv`, 也不接入真实硬件.

### 一、先确定异常状态的行为

![[Pasted image 20261120165859.png]]



| 事件              | 作用                  |
| --------------- | ------------------- |
| FAULT_RAISED    | 触发故障, 进入FAULT       |
| FAULT_CLEARED   | 模拟故障原因已经消除          |
| RESET_FAULT     | 满足条件后, 从FAULT返回INIT |
| ESTOP_TRIGGERED | 触发急停, 进入到ESTOP      |
| ESTOP_RELEASED  | 模拟急停输入已经解除          |
| RESET_ESTOP     | 满足条件后, 从ESTOP返回INIT |
这里采用三个基本原则:
- 1 . ESTOP 优先级高于FAULT . 即使急停期间又发生了故障, 整机状态仍保持ESTOP, 但要记录故障
- 2. 解除异常不等于复位. 例如`ESTOP_RELEASED` 不会自动返回INIT, 必须再次执行`RESET_ESTOP`
- 3. 复位只能返回INIT. 不允许故障或急停解除后直接恢复到之前的MANUAL、AUTO


### 二、 修改system_manager_node.py

这次只修改三个位置.

1 、 扩展状态转换表
找到现有`TRANSITIONS` , 完整替换为:
```
TRANSITIONS = {
	# 初始化 
	(SystemState.INIT, "INIT_SUCCESS"):SystemState.IDLE,
	
	# 手动模式
	(SystemState.IDLE, "START_MANUAL"):SystemState.MANUAL,
	(SystemState.MANUAL,"STOP"):SystemState.IDLE,
	
	# 自动模式
	(SystemState.IDLE, "START_AUTO"):SystemState.AUTO
	(SystemState.AUTO, "STOP"):SystemState.IDLE,
	(SystemState.AUTO,"TASK_FINISHED"):SystemState.IDLE,
	
	# 异常复位
	(SystemState.FAULT,"RESET_FAULT"):SystemState.INIT,
	(SysteState.ESTOP,"RESET_ESTOP"):SystemState.INIT,
	
}
```
这里没有将`FAULT_RAISED` 和 `ESTOP_TRIGGERED` 写成普通状态转换表规则, 因为它们属于全局异常事件, 需要从多个状态触发.

2、增加异常输入标志
找到`SystemManager.__init__()` 中的 `self._state=SystemState.INIT`, 在其后面添加:
```
# 模拟的外部异常输入状态
self._fault_input_active = False
self._estop_input_active = False
```
上述与FSM状态不是同一个概念, 例如机器人已经进入FAULT, 即使`_fault_input_active`变成False, 也必须经过明确复位才能离开FAULT

3、替换`_try_transition()`
将原来的`_try_transition()` 方法完整替换成:
```
def _try_transition(self,event):
	previous = self._state
	# 1. 急停触发, 最高优先级
	if event == "ESTOP_TRIGGERED":
		self._estop_input_active = True
		target = SystemState.ESTOP
	# 2. 故障触发
	elif event == "FAULT_RAISED":
		self._fault_input_active = True
		
		# 急停期间发生故障, 不覆盖ESTOP状态
		target = (
			SystemState.ESTOP,
			if self._state == SystemState.ESTOP
			else SystemState.FAULT
		)
	# 3. 模拟急停输入解除
	elif event == "ESTOP_RELEASED":
		if(self._state != SystemState.ESTOP or note self._estop_input_active):
			return False, "Rejected: no active ESTOP input"
		self._estop_input_active = False
		return True, "ESTOP released; explicit reset required"
	# 4. 模拟故障输入消除
	elif event == "FAULT_CLEARED":
		if not self._fault_input_active:
			return False, "Rejected: no active fault input"
		self._fault_input_active = False
		return True, "Fault cleared: explicit reset required"
	# 5. 普通状态转换及复位
	else:
		target = TRANSITIONS.get((self._state, event))
		if target is None:
			return False, (f"Rejected: {self._state.value}+{event}")
		# 故障尚未清楚, 禁止复位
		if(event == "RESET_FAULT" and self._fault_input_active):
			return False, "Rejected: fault input still active"

		# 急停未解除或仍存在故障, 禁止复位
		if event == "RESET_ESTOP":
			if(self._estop_input_active or self._fault_input_active):
				return False, (f"Rejected: ESTOP or fault input still active")
	# 6. 转换未改变, 避免重复发布
	if target == previous:
		return True,(
				f"Accepted: {event}; " 
				f"state remains {previous.value}"
			)
	# 7. 提交状态变更
	self._state = target
	self.get_logger().info(
		f"FSM: {previous.value} -> {target.value}, "
		f"event={event}"
	)
	self._publish_state()
	return True, self._state.value

```
这样处理之后, 异常事件与普通状态切换仍统一经过`_try_transition()`

4、小幅调整Service返回原因

这部分属于第三处修改中的配套调整: 在现有`_handle_send_event()`中, 找到:
```
if accepted:
	response.reason = (
		f"Accepted: {previous} -> {self._state.value}"
	)
else:
	response.reason = result
```
替换为:
```
if accepted and previous != self._state.value
	response.reason = (
		f"Accepted: {previous} -> {self._state.value}"
	)
else:
	response.reason = result
```
这是为了区分两类事件:
- `FAULT_RAISED` :  可能真正发生状态转换
- `FAULT_CLEARED`:  只是异常输入解除, 整机状态暂时不变
原来的`/system/get_state`、`/system/state` 、`/system/mock_init_success` 全部保持不变

### 三、重新构建
停止旧节点, 执行:
```
cd ~/robot_foundation_ws 
source /opt/ros/jazzy/setup.bash

colcon build --packages-select robot_system_manager --symlink-install

source install/setup.bash

ros2 run robot_system_manager system_manager_node
```
预期输出:
```
System state: INIT
```

### 四、测试FAULT异常流程

开启新的终端, 执行指令:
```
source ~/robot_foundation_ws/install/setup.bash
```

为了方便后续反复测试, 先在当前终端定义一个小函数:
```
send_event(){
	ros2 service call /system/send_event robot_interfaces/srv/SendEvent "{event: $1, source: cli_test}"
}
```

依次执行:
```
# 初始化完成
send_event INIT_SUCCESS

# 进入自动模式
send_event START_AUTO

# 模拟发生故障
send_event FAULT_RAISED
```

此时应该进入:
```
AUTO -> FAULT
```
接着尝试在故障尚未清除时复位:
```
send_event RESET_FAULT
```

预期:
```
accepted=False
current_state='FAULT'
reason='Rejected: fault input still active'
```
然后模拟故障消除:
```
send_event FAULT_CLEARED
```
再次请求复位:
```
send_event RESET_FAULT
```
预期: 
```
accepted=True
previous_state='FAULT'
current_state='INIT'
```
这就验证了完整的故障触发 ->  故障消除 -> 人工请求复位流程


### 五、测试ESTOP急停优先级

继续使用当前终端执行以下指令:
```
send_event INIT_SUCCESS
send_event START_MANUAL
send_event ESTOP_TRIGGERED
```

此时应该进入到ESTOP状态.
```
waiting for service to become available...
requester: making request: robot_interfaces.srv.SendEvent_Request(event='INIT_SUCCESS', source='cli_test')

response:
robot_interfaces.srv.SendEvent_Response(accepted=True, previous_state='INIT', current_state='IDLE', reason='Accepted: INIT -> IDLE')

waiting for service to become available...
requester: making request: robot_interfaces.srv.SendEvent_Request(event='START_MANUAL', source='cli_test')

response:
robot_interfaces.srv.SendEvent_Response(accepted=True, previous_state='IDLE', current_state='MANUAL', reason='Accepted: IDLE -> MANUAL')

waiting for service to become available...
requester: making request: robot_interfaces.srv.SendEvent_Request(event='ESTOP_TRIGGERED', source='cli_test')

response:
robot_interfaces.srv.SendEvent_Response(accepted=True, previous_state='MANUAL', current_state='ESTOP', reason='Accepted: MANUAL -> ESTOP')
```
现在故意再触发一个故障:
```
send_event FAULT_RAISED
```
预期accepted =  True, 但状态继续保持:
```
waiting for service to become available...
requester: making request: robot_interfaces.srv.SendEvent_Request(event='FAULT_RAISED', source='cli_test')

response:
robot_interfaces.srv.SendEvent_Response(accepted=True, previous_state='ESTOP', current_state='ESTOP', reason='Accepted: FAULT_RAISED state remains ESTOP')
```
这说明ESTOP不会被普通故障覆盖, 而故障输入也被记录了, 接下来模拟急停解除:
```
send_event ESTOP_RELEASED
```

```
waiting for service to become available...
requester: making request: robot_interfaces.srv.SendEvent_Request(event='ESTOP_RELEASED', source='cli_test')

response:
robot_interfaces.srv.SendEvent_Response(accepted=True, previous_state='ESTOP', current_state='ESTOP', reason='ESTOP released; explicit reset required')
```

此时直接复位:
```
send_event RESET_ESTOP
```

应该被拒绝
```
waiting for service to become available...
requester: making request: robot_interfaces.srv.SendEvent_Request(event='RESET_ESTOP', source='cli_test')

response:
robot_interfaces.srv.SendEvent_Response(accepted=False, previous_state='ESTOP', current_state='ESTOP', reason='Rejected: ESTOP or fault input still active')
```
因为刚才触发的故障还没有清除, 最后执行
```
send_event FAULT_CLEARED
send_event RESET_ESTOP
```

```
waiting for service to become available...
requester: making request: robot_interfaces.srv.SendEvent_Request(event='FAULT_CLEARED', source='cli_test')

response:
robot_interfaces.srv.SendEvent_Response(accepted=True, previous_state='ESTOP', current_state='ESTOP', reason='Fault cleared; explicit reset required')

waiting for service to become available...
requester: making request: robot_interfaces.srv.SendEvent_Request(event='RESET_ESTOP', source='cli_test')

response:
robot_interfaces.srv.SendEvent_Response(accepted=True, previous_state='ESTOP', current_state='INIT', reason='Accepted: ESTOP -> INIT')
```
至此完成急停与故障并发情况下的优先级、锁存和复位验证.
