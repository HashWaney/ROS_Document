
目前`robot_system_manager` 已经具备六状态FSM、通用事件接口、异常复位、事件日志和自动化回归测试.
本次只增加一个功能: **根据事件类型检查其声明的来源是否符合规则, 拒绝来源不匹配的事件.**
不新增Package、不修改`SendEvent.srv`, 也不改变已经验收的状态转换规则.

这里先明确一个技术边界: `SendEvent.srv` 中的source 是调用方自行填写的字符串, 可以被伪造. 本步骤实现的是业务层来源规则校验, 并非真正的身份认证. 真实的ROS 2 通信身份认证与访问控制需要另外引入SROS 2/ DDS-Security


### 一、定义事件来源规则
先为当前系统定义几个逻辑来源

| 事件                           | 允许的source      |
| ---------------------------- | -------------- |
| INIT_SUCCESS                 | self_check     |
| START_MANUAL/START_AUTO      | operator       |
| TASK_FINISHED                | task_executor  |
| FAULT_CLEARED                | fault_manager  |
| ESTOP_RELEASED               | safety_monitor |
| RESET_FAULT/RESET_ESTOP      | operator       |
| FAULT_RAISED/ESTOP_TRIGGERED | 不限制来源          |
| STOP                         | 不限制来源          |
为什么有三个事件不限制来源?
因为STOP、FAULT_RAISED、ESTOP_TRIGGERED 都是趋向停止运行的事件. V1不应因为来源标识填写错误, 就阻止软件层进入停止或异常状态, 反过来, 解除故障、解除急停、恢复运行必须经过来源规则检查.
这只是当前的业务接口策略, 等我们真正接入硬件安全机制时, 急停及运动禁止不能依赖这些ROS 2 Service


### 二、修改system_manager_node.py

1、添加事件来源规则表
在现有`TRANSITIONS` 定义后, 添加:
```
# V1 逻辑来源规则(不是身份认证)
EVENT_SOURCE_RULES = {

	"INIT_SUCCESS": {"self_check"},
	"START_MANUAL": {"operator"},
	"START_AUTO": {"operator"},
	"TASK_FINISHED": {"task_executor"},
	
	"FAULT_CLEARED": {"fault_manager"},
	"ESTOP_RELEASED": {"safety_monitor"},
	
	"RESET_FAULT": {"operator"},
	"RESET_ESTOP": {"operator"},
}


# 停止、故障和急停触发事件不限制来源
UNRESTRICTED_EVENTS = {
	"STOP",
	"FAULT_RAISED",
	"ESTOP_TRIGGERED",
}

```
这里只添加业务来源, 不改变之前的`TRANSITIONS`

2、 新增来源校验函数
在`SystemManager` 类中添加:
```
def _check_event_source(self,event,source):
	# 趋向停止的事件不受来源限制
	if event is UNRESTRICTED_EVENTS:
		return True, ""
	allowed_sources = EVENT_SOURCE_RULES.get(event)
	# 未定义的事件仍交给FSM 统一拒绝
	if allowed_sources is None:
		return True,""
	if source in allowed_sources:
		return True, ""
		
	reason = (
		f"Rejected: source '{source}' "
		f"not allowed for event '{event}'"
	)
	return False, reason
```

它只处理来源规则, 不负责真正修改状态
例如:
```
INIT_SUCCESS + operator
	-> 来源校验失败
	-> 不执行INIT-> IDLE
```
而:
```
INIT_SUCCESS + self_check
	-> 来源规则通过
	-> 继续交给FSM 检查状态转换是否合法
```
因此, 即使来源符合要求, 也不能绕过现有FSM转换规则

3、统一处理事件
将原来的`_handle_send_event()`完整替换成:
```
def _handle_send_event(self,request,response):
	event = request.event.strip().upper()
	source = request.source.strip() or "unknown"
	pervious = self._state.value
	
	# 第一层: 来源声明校验
	allowed, reason = self._check_event_source(
		event, source
	)
	if not allowed:
		accepted = False
		result = reason 
	else:
		# 第二层: 原有FSM 状态转换检查
		accepted, result = self._try_transition(event)
		
	response.accepted = accepted
	response.previous_state = previous
	response.current_state = self._state.value
	
	if accepted and previous != self._state.value
		response.reason = (
			f"Accepted: {previous}->{self._state.value}"
		)
	else:
		response.reason = result
	# 来源校验失败的事件同样记录
	self._publish_state_event(
		event=event,
		source=source,
		previous_state=response.previous_state,
		current_state=response.current_state,
		accepted=response.accepted,
		reason=response.reason
	)
	self.get_logger().info(
		f"Event={event}, source={source}, "
		f"accepted={accepted},"
		f"state={response.current_state}"
	
	)
	return response
```