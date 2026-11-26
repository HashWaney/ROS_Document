
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

创建Package:
```
cd ~/robot_foundation_ws/src
ros2 pkg create robot_lifecycle_manager \
	--build-type ament_python \
	--license Apache-2.0 \
	--dependencies rclpy lifecycle_msgs std_msgs std_srvs robot_interfaces
```

### 四、实现第一个Lifecycle Node
以模拟传感器节点为例, 让它在`Active`状态下每秒发布一次模拟数据, 在`Inactive` 状态下停止发布. 

创建sensor节点, 进入到`/src/robot_lifecycle_manager/robot_lifecycle_manager/`, 新建sensor_node.py
```
import rclpy

from rclpy.lifecycle import (LifecycleNode,TransitionCallbackReturn)

from std_msgs.msg import String

class SensorNode(LifecycleNode):
    
    def __init__(self):
        super().__init__("sensor_node")
        self._pub = None
        self._timer = None
        self._count =0 
        self.get_logger().info("SensorNode created")

    def on_configure(self,state):
        self.get_logger().info("on_configure")

        self._pub = self.create_lifecycle_publisher(
            String,
            "/sensor/data",
            10
        )
        self._count=0
        return TransitionCallbackReturn.SUCCESS
    
    def on_activate(self, state):
        self.get_logger().info("on_activate")
        result = super().on_activate(state)
        if result != TransitionCallbackReturn.SUCCESS:
            return result
        self._timer = self.create_timer(1.0,self._publish_data)
        return TransitionCallbackReturn.SUCCESS
    
    def on_deactivate(self, state):
        self.get_logger().info("on_deactive")
        self._stop_timer()
        return super().on_deactivate(state)
    
    def on_cleanup(self, state):
        self.get_logger().info("on_cleanup")
        self._release_resources()
        return TransitionCallbackReturn.SUCCESS

    def on_shutdown(self, state):
        self.get_logger().info("on_shutdown")
        self._release_resources()
        return TransitionCallbackReturn.SUCCESS
    
    def _publish_data(self):
        msg = String()
        msg.data = f"Sensor_data_{self._count}"
        self._pub.publish(msg)
        self._count +=1

    def _stop_timer(self):
        if self._timer is not None:
            self.destroy_timer(self._timer)
            self._timer=None

    def _release_resources(self):
        self._stop_timer()
        if self._pub is not None:
            self.destroy_lifecycle_publisher(
                self._pub
            )
            self._pub = None
        self._count =0 

def main(args=None):
    rclpy.init(args=args)

    node = SensorNode()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        if rclpy.ok():
            rclpy.shutdown()

if __name__ == "__main__":
    main()
        
```
与之前的`SystemManager(Node)` 不同, 这次继承的是`LifecycleNode`,因此ROS 2 会自动提供标准生命周期管理接口, 另外使用的是`create_lifecycle_publisher(...)` 而不是普通的`create_publisher()`, 在`on_activate()` 和`on_deactivate()`中调用父类方法, 可以让Lifecycle Publisher正确切换启用状态.


### 五、实现Lifecycle Manager
现在Sensor Node 已经具备生命周期管理能力, 但它不会主动执行`configure` 或 `activate`, 因此需要一个管理节点来协调这些操作.
ROS 2 提供以下标准接口:

| 接口                            | 类型                                 |
| ----------------------------- | ---------------------------------- |
| /sensor_node/get_state        | lifecycle_msgs/srv/GetState        |
| /sensor_node/change_state     | lifecycle_msgs/srv/ChangeState     |
| /sensor_node/transition_event | lifecycle_msgs/srv/TransitionEvent |

这次的管理节点通过这些接口控制Sensor Node, 不直接调用Sensor Node 的Python方法, 本次先采用 **一次性执行的管理节点**, 通过`operation` 参数选择启动或停用操作, 这样不需要额外引入多线程、异步状态编排和复杂的Executor管理.
创建lifecycle_manager_node节点, 新建`~/src/robot_lifecycle_manager/robot_lifecycle_mangaer/lifecycle_manager_node.py`
```
import rclpy
from rclpy.node import Node
from lifecycle_msgs.msg import State, Transition
from lifecycle_msgs.srv import (GetState, ChangeState)

from std_srvs.srv import Trigger
from robot_interfaces.srv import SendEvent

class RobotLifecycleManager(Node):
    def __init__(self):
        super().__init__("robot_lifecycle_manager")

        self.declare_parameter("operation","startup")

        self._get_state_client = self.create_client(
            GetState,
            "/sensor_node/get_state"
        )
        self._change_state_client = self.create_client(
            ChangeState,
            "/sensor_node/change_state"
        )
        self._system_state_client = self.create_client(
            Trigger,
            "/system/get_state"
        )

        self._event_client = self.create_client(
            SendEvent,
            "/system/send_event" # 发送事件给system_manager_node节点， 在该节点的_handle_send_event方法处理
        )
    def _call_service(self,client,request):
        if not client.wait_for_service(timeout_sec=5.0):
            raise RuntimeError(
                f"Service unavailable: {client.srv_name}"
            )
        future = client.call_async(request)
        rclpy.spin_until_future_complete(
            self,
            future,
            timeout_sec=5.0
        )
        if not future.done():
            raise RuntimeError(
                f"Service timeout: {client.srv_name}"
            )
        return future.result()
    
    def get_sensor_state(self):
        response = self._call_service(
            self._get_state_client,
            GetState.Request()
        )
        return response.current_state
    def change_sensor_state(self,transition_id):
        request = ChangeState.Request()
        request.transition.id = transition_id

        response = self._call_service(
            self._change_state_client,
            request 
        )
        if not response.success:
            raise RuntimeError(
                f"Transition failed: {transition_id}"
            )
        state = self.get_sensor_state()
        self.get_logger().info(
            f"Sensor state: {state.label}"
        )
        return state.id
    def send_system_event(self,event,source):
        request = SendEvent.Request()
        request.event = event
        request.source = source

        response = self._call_service(
            self._event_client,
            request
        )

        if not response.accepted:
            raise RuntimeError(response.reason)

        self.get_logger().info(
            f"System state: {response.current_state}"
        )

    def startup(self):
        # 确认整机处于初始化阶段
        response = self._call_service(
            self._system_state_client,
            Trigger.Request()
        )
        if not response.success or response.message !="INIT":
            raise RuntimeError(
                "System must be in Init"
            )
        try:
            state = self.get_sensor_state().id
            # 未配置 ---》 已配置
            if state == State.PRIMARY_STATE_UNCONFIGURED:
                state = self.change_sensor_state(Transition.TRANSITION_CONFIGURE)

            # 已配置 ---》 激活
            if state == State.PRIMARY_STATE_INACTIVE:
                state = self.change_sensor_state(Transition.TRANSITION_ACTIVATE)

            # 确认真正进入Active
            if state != State.PRIMARY_STATE_ACTIVE:
                raise RuntimeError(
                    "Sensor node is not active"
                )

            # 通知整机FSM 初始化完成
            self.send_system_event("INIT_SUCCESS","self_check")
            self.get_logger().info("Robot startup completed")

        except Exception:
              # 启动失败，尝试通知整机FSM
            try:
                self.send_system_event(
                    "FAULT_RAISED",
                    "lifecycle_manager"
                )
            except Exception as error:
                self.get_logger().info(
                    f"Fault reporting failed :{error}"
                )
            raise
    def stop(self):
        state = self.get_sensor_state().id

        # 本阶段将必要模块停用视为不可运行故障
        try:
            self.send_system_event(
                "FAULT_RAISED",
                "lifecycle_manager"             
            )

        finally:
            if state == State.PRIMARY_STATE_ACTIVE:
                state = self.change_sensor_state(
                    Transition.TRANSITION_DEACTIVATE
                )
            if state != State.PRIMARY_STATE_INACTIVE:
                raise RuntimeError(
                    "Sensor node is not inactive"
                )
        self.get_logger().info("Sensor node stopped")

def main(args=None):
    rclpy.init(args=args)

    node = RobotLifecycleManager()

    exit_code =0

    try: 
        operation = node.get_parameter(
            "operation"
        ).value
        if operation == "startup": # 启动生命周期管理节点， 通过调用该节点的startup方法，开启sensor节点的配置相关信息 然后触发sensor节点相应的生命周期函数回调
            node.startup()

        elif operation =="stop":
            node.stop()
        else:
            raise RuntimeError(
                f"Unknown operation: {operation}"
            )
    except Exception as error:
        node.get_logger().error(str(error))
        exit_code = 1 
    finally:
        node.destroy_node()
        if rclpy.ok():
            rclpy.shutdown()
    return exit_code

if __name__ == "__main__":
    raise SystemExit(main())
```

这里重点理解三个方法:

| 方法                      | 职责           |
| ----------------------- | ------------ |
| `get_sensor_state()`    | 查询节点真实生命周期状态 |
| `change_sensor_state()` | 请求节点执行生命周期转换 |
| `send_system_event()`   | 向整机FSM上报系统事件 |
`_call_service()`中使用`call_async()` 和 有限时间等待, 避免Service 不存在时无限等待, 这里的等待发生在`main()`调用链中, 而不是ROS 2 回调内部;另外，因为当前 FSM 尚未定义维护模式，本次将必要模块停用处理为 `FAULT_RAISED`。这是本阶段的简化策略，不代表生产环境中所有主动停机都应该被视为故障。

### 六、注册节点并构建
修改`setup.py`
```
entry_points = {
	'console_scripts': [
		'sensor_node = robot_lifecycle_manager.sensor_node:main',
		'lifecycle_manager_node = robot_lifecycle_manager.lifecycle_manager_node:main',
	]
}
```
构建
```
source /opt/ros/jazzy/setup.bash

colcon build --packages-up-to robot_lifecycle_manager --symlink-install

source install/setup.bash
```

### 七、运行与整体架构验收

**实验A: 理解原生Lifecycle**

1、 终端A启动Sensor Node:

```
ros2 run robot_lifecycle_manager sensor_node
```
2、终端B执行:
```
source install/setup.bash

ros2 lifecycle nodes

ros2 lifecycle get /sensor_node
```

预期输出:
```
unconfigured [1]
```

然后依次操作:
```
ros2 lifecycle set /sensor_node configure 

ros2 lifecycle get /sensor_node

ros2 lifecycle set /sensor_node activate

ros2 lifecycle get /sensor_node
```
预期输出:
```
inactive [2]

active [3]
```

3、终端C订阅
```
ros2 topic echo /sensor/data
```

应该持续收到:
```
data: sensor_data_0
---
data: sensor_data_1
---
data: sensor_data_2
```
4、回到终端B
```
ros2 lifecycle set /sensor_node deactivate

ros2 lifecycle set /sensor_node cleanup
```
此时数据停止发布、节点重新回到`Unconfigured`, 这就完成了一次完整的节点生命周期循环

**实验B: Lifecycle与整机FSM联动**

保持Sensor Node 运行在`Unconfigured` 状态. 另外打开终端, 启动之前的System Manager:
```
source install/setup.bash

ros2 run robot_system_manager system_manager_node
```
整机初始状态应该是`INIT`, 现在执行管理节点:
```
ros2 run robot_lifecycle_manager lifecycle_manager_node
```
整体调用过程如下:
![[Step4-node.svg]]
管理节点预期输出以下内容:
```
Sensor state: inactive 
Sensor state: active
System state: IDLE
Robot startup completed
```
此时Lifecycle Manager 进程正常退出, 但Sensor Node 和 SystemManager仍然持续运行, 查询两种状态:
```
ros2 lifecycle get /sensor_node

ros2 service call /system/get_state std_srvs/srv/Trigger '{}'
```

预期输出:
```
hash@ros:~/robot_foundation_ws$ ros2 lifecycle get /sensor_node
active [3]
hash@ros:~/robot_foundation_ws$ ros2 service call /system/get_state std_srvs/srv/Trigger '{}'
waiting for service to become available...
requester: making request: std_srvs.srv.Trigger_Request()

response:
std_srvs.srv.Trigger_Response(success=True, message='IDLE')

```

至此, 第一次打通了两个独立状态管理zhi jai