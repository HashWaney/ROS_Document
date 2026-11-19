
本次只完成一个完整的Step: **创建整机状态管理节点, 让它能够启动,维护当前状态,并通过ROS 2 Service查询状态**, 暂时不接入行为树、运动控制、网络通信和故障管理


### 一、本次的最小架构

![[Pasted image 20261119143805.png]]

目前先保留已确认的六个整机状态:
`INIT` 、`IDLE` 、`MANUAL`、`AUTO` 、`FAULT`、`ESTOP`, 本步骤只实现`INIT`初始状态和状态查询.  状态转换逻辑放在下一步; 技术选择采用
`Python + rclpy + ament_python`, 先验证状态机架构. 理由是状态管理属于低频控制逻辑, 首阶段没有硬实时计算要求; 使用Python可以更快验证状态流转和服务接口. 将来需要与C++运动节点通信时, 直接使用标准ROS 2消息即可.

接口约定:

| 接口                  | 类型                     | 作用     |
| ------------------- | ---------------------- | ------ |
| `/system/state`     | `std_msgs/msg/String`  | 发布整机状态 |
| `/system/get_state` | `std_srvs/srv/Trigger` | 查询当前状态 |

### 二、创建ROS 2 Package

在Ubuntu终端执行:
```
cd ~/robot_foundation_ws
source /opt/ros/jazzy/setup.bash

# 核验所需依赖
ros2 pkg prefix rclpy
ros2 pkg prefix std_msgs
ros2 pkg prefix std_srvs
```
三项都能找到后, 创建Package, 执行下列指令:
```
cd ~/robot_foundation_ws/src

ros2 pkg create robot_system_manager \
	--build-type ament_python \
	--license Apache-2.0 \
	--dependencies rclcpy std_msgs std_srvs
```

预期目录:
```
robot_foundation_ws/src/
├── robot_description/
└── robot_system_manager/
    ├── package.xml
    ├── setup.py
    ├── setup.cfg
    └── robot_system_manager/
        └── __init__.py
```
这里采用独立Package, 避免把系统状态管理逻辑放进`robot_description`

### 三、实现最小状态管理节点

创建文件:
```
vi ~/robot_foundation_ws/src/robot_system_manager/robot_system_manager/system_manager_node.py
```

写入:
```

from enum import Enum

import rclpy
from rclpy.node import Node
from rclpy.qos import QoSProfile, DurabilityPolicy
from std_msgs.msg import String
from std_srvs.srv import Trigger


class SystemState(str, Enum):
    INIT = "INIT"
    IDLE = "IDLE"
    MANUAL = "MANUAL"
    AUTO = "AUTO"
    FAULT = "FAULT"
    ESTOP = "ESTOP"


class SystemManager(Node):

    def __init__(self):
        super().__init__("robot_system_manager")

        # 当前整机状态唯一由本节点维护
        self._state = SystemState.INIT

        # 保留最新状态，供后加入的订阅者获取
        qos = QoSProfile(
            depth=1,
            durability=DurabilityPolicy.TRANSIENT_LOCAL
        )

        self._state_pub = self.create_publisher(
            String, "/system/state", qos
        )

        self._get_state_srv = self.create_service(
            Trigger,
            "/system/get_state",
            self._handle_get_state
        )

        self._publish_state()

    def _publish_state(self):
        msg = String()
        msg.data = self._state.value
        self._state_pub.publish(msg)
        self.get_logger().info(f"System state: {msg.data}")

    def _handle_get_state(self, request, response):
        response.success = True
        response.message = self._state.value
        return response


def main(args=None):
    rclpy.init(args=args)
    node = SystemManager()

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

这段代码目前只有三个职责: 定义六个状态、启动时初始化为INIT、对外提供查询接口.

### 四、注册可执行节点

打开`/robot_foundation_ws/src/robot_system_manager/setup.py` , 找到`entry_points`, 将`console_scripts` 设置为:
```
entry_points = {
	'console_scripts': [
		'system_manager_node = robot_system_manager.system_manager_node:main'
	]
}
```
其他配置保持不变;

### 五、构建并验证

> 1 、构建:
```
cd ~/robot_foundation_ws

source /opt/ros/jazzy/setup.bash

colcon build --packages-select robot_system_manager --symlink-install

source install/setup.bash
```

> 2 、启动FSM节点

在终端A运行system_manager_node 节点
```
ros2 run robot_system_manager system_manager_node
```

预期输出日志:
```
[INFO] [1795074896.591389434] [robot_system_manager]: System state: INIT
```

> 3、查询整机状态

在终端B执行以下指令:
```
source ~/robot_foundation_ws/install/setup.bash

ros2 service call /system/get_state std_srvs/srv/Trigger '{}'
```
预期返回:
```
response:
std_srvs.srv.Trigger_Response(success=True, message='INIT')
```

> 4、 再验证Topic
 
 终端A: 保持`robot_system_manager`运行, 在终端B中执行
```
source ~/robot_foundation_ws/install/setup.bash

ros2 topic echo /system/state std_msgs/msg/String \
  --qos-reliability reliable \
  --qos-durability transient_local 
```

预期会输出:
```
data: INIT
```
> 5、检查订阅