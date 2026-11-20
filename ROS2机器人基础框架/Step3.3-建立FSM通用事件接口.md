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

### 三、修改robot_system_manager

1、导入新接口
在文件顶部添加:
```
from robot_interfaces.srv import SendEvent
```

2、注册通用事件Service
在`SystemManager.__init__()` 中添加:
```
self._send_event_srv = self.create_service(SendEvent, "/system/send_event",self._handle_send_event)
```
之前的`/system/mock_init_success` 可以暂时保留, 方便回归测试. 两个接口都通过`_try_transition()`修改状态, 不建立第二套FSM逻辑

3、添加事件处理函数
在`SystemManager`类中添加:
```
def _handle_send_event(self,request,response):
	event = request.event.strip().upper()
	source = request.source.strip() or "unknown"
	
	previous = self._state.value
	accepted, result = self._try_transition(event)
	response.accepted = accepted 
	response.previous_state = previous
	response.current_state=self._state.value
	if accepted:
		response.reason = (
			f"Accepted: {previous} -> {self._state.value}"
		)
	else:
		response.reason = result
	
	self.get_logger().info(
		f"Event ={event}, source={source}, "
		f"accepted={accepted} , "
		f"state={response.current_state}"
		
	)
	return response
	
```
这里复用Step3.2 已经完成的`_try_transition()` , 暂时不添加新的状态转换规则.  因此目前通用接口只接受:
```
INIT + INIT_SUCCESS -> IDLE
```
其他尚未实现的事件统一拒绝.

4、声明Package依赖
打开src/robot_system_manager/package.xml, 在依赖区域添加:
```
<exec_depend>robot_interfaces</exec_depend>
```

### 四、构建并运行

先停止旧的`robot_system_manager` 节点, 然后执行以下指令
```
cd ~/robot_foundation_ws

source /opt/ros/jazzy/setup.bash

colcon build --packages-up-to robot_system_manager --symlink-install

source install/setup.bash
```

验证自定义接口是否生成:
```
ros2 interface show robot_interfaces/srv/SendEvent
```
应能看到刚才定义的请求和响应字段

启动节点, 此时状态应该为INIT
```
ros2 run robot_system_manager system_manager_node
```

### 五、验证通用事件接口

新开终端并加载工作空间
```
source ~/robot_foundation_ws/install/setup.bash
```

1 、测试合法转换:
```
ros2 service call /system/send_event robot_interfaces/srv/SendEvent "{event: INIT_SUCCESS, source: cli_test}"
```
预期关键字段:
```
requester: making request: robot_interfaces.srv.SendEvent_Request(event='INIT_SUCCESS', source='cli_test')

response:
robot_interfaces.srv.SendEvent_Response(accepted=True, previous_state='INIT', current_state='IDLE', reason='Accepted: INIT -> IDLE')
```


2、 测试重复事件
再次发送同一个事件:
```
ros2 service call /system/send_event robot_interfaces/srv/SendEvent "{event: INIT_SUCCESS, source: cli_test}"
```
预期输出:
```
requester: making request: robot_interfaces.srv.SendEvent_Request(event='INIT_SUCCESS', source='cli_test')

response:
robot_interfaces.srv.SendEvent_Response(accepted=False, previous_state='IDLE', current_state='IDLE', reason='Rejected: IDLE+ INIT_SUCCESS')
```

3、测试为定义事件:
```
ros2 service call /system/send_event robot_interfaces/srv/SendEvent "{event: START_AUTO, source: cli_test}"
```
由于尚未实现IDLE  -> AUTO 规则, 应该返回:
```
requester: making request: robot_interfaces.srv.SendEvent_Request(event='START_AUTO', source='cli_test')

response:
robot_interfaces.srv.SendEvent_Response(accepted=False, previous_state='IDLE', current_state='IDLE', reason='Rejected: IDLE+ START_AUTO')

```

4、查询当前状态
```
ros2 service call /system/get_state std_srvs/srv/Trigger '{}'
```
预期输出:
```
requester: making request: std_srvs.srv.Trigger_Request()

response:
std_srvs.srv.Trigger_Response(success=True, message='IDLE')
```