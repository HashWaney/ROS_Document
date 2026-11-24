
目前已经通过ROS 2 CLI 手动验证了状态转换、故障锁存、急停优先级和结构化事件记录.
现在的问题是: 以后修改FSM代码, 怎么快速发现是否破坏了之前已经验证的规则?
本次只实现一个完整的功能: 使用`pytest` +` colcon test`  建立FSM自动化回归测试.


一、测试方案
![[Pasted image 20261123175639.png]]

这次属于节点内部逻辑测试, 不是ROS 2 Service、Topic 或 DDS的端到端通信测试. 通信链路此前已经通过手动验证, 暂不需要重复实现. 计划覆盖以下7类测试

| 测试  | 验证内容                      |
| --- | ------------------------- |
| 1   | 节点初始化状态为INIT              |
| 2   | INIT_SUCCESS 成功, 重复事件被拒绝  |
| 3   | MANUAL和AUTO不允许直接切换        |
| 4   | AUTO收到TASK_FINISHED返回IDLE |
| 5   | FAULT必须先清除故障才能复位          |
| 6   | ESTOP优先级及复位约束             |
| 7   | 未定义事件被拒绝                  |
ROS 2 Jazzy 支持在`ament_python` Package中使用`pytest`, 并通过`colcon test` 执行

### 二、Gate0 : 检查测试环境
执行下列指令:
```
source install/setup.bash

python3 -m pytest --version
```
如果出现pytest版本号, 就可以继续, 如果提示没有安装:
```
sudo apt update 
sudo apt install python3-pytest
```

### 三、创建FSM测试文件
保持现有工程目录不变, 只新增一个文件
```
robot_foundation_ws/
└── src/
    └── robot_system_manager/
        ├── robot_system_manager/
        │   └── system_manager_node.py
        ├── test/
        │   └── test_fsm.py       ← 本次新增
        ├── setup.py
        └── package.xml
```

创建测试目录:
```
cd ~/robot_foundation_ws/src/robot_system_manager

mkdir -p test

vi test/test_fsm.py
```

将下面的代码完整写入:
```
"""Regression tests for robot_system_manager FSM."""

import pytest
import rclpy

from robot_interfaces.srv import SendEvent
from robot_system_manager.system_manager_node import SystemManager

@pytest.fixture
def fsm():
    """Create an isolated FSM node for each test."""
    rclpy.init()
    node = None
    try:
        node = SystemManager()
        yield node
    finally:
        if node is not None:
            node.destroy_node()
        if rclpy.ok():
            rclpy.shutdown()

def send(node, event):
    """Submit an event through the existing service handler."""
    request = SendEvent.Request()
    request.event = event 
    request.source = "pytest"
    response = SendEvent.Response()
    return node._handle_send_event(request,response)

def test_fsm_initial_state(fsm):
    assert fsm._state.value =="INIT"

def test_fsm_init_and_duplicate(fsm):
    result = send(fsm,"INIT_SUCCESS")

    assert result.accepted is True
    assert result.previous_state == "INIT"
    assert result.current_state == "IDLE"

    duplicate = send(fsm,"INIT_SUCCESS")
    assert duplicate.accepted is False
    assert duplicate.current_state == "IDLE"

def test_fsm_mode_switch(fsm):
    send(fsm,"INIT_SUCCESS")

    result = send(fsm,"START_MANUAL")

    assert result.accepted is True
    assert result.current_state =="MANUAL"

    # MANUAL cannot switch directly to AUTO
    result = send(fsm,"START_AUTO")
    assert result.accepted is False
    assert result.current_state == "MANUAL"

    result = send(fsm,"STOP")
    assert result.current_state =="IDLE"

    result = send(fsm,"START_AUTO")
    assert result.accepted is True
    assert result.current_state == "AUTO"

    # Auto cannot switch directly to manual 
    result = send(fsm,"START_MANUAL")
    assert result.accepted is False
    assert result.current_state =="AUTO"


def test_fsm_task_finished(fsm):
    send(fsm,"INIT_SUCCESS")
    send(fsm,"START_AUTO")

    result = send(fsm,"TASK_FINISHED")
    assert result.accepted is True
    assert result.current_state == "IDLE"

def test_fsm_fault_reset(fsm):
    send(fsm,"INIT_SUCCESS")
    send(fsm,"START_AUTO")

    result = send(fsm,"FAULT_RAISED")
    assert result.current_state =="FAULT"

    # Fault is still active
    result = send(fsm,"RESET_FAULT")
    assert result.accepted is False
    assert result.current_state =="FAULT"

    # Clearing the fault must not be reset the FSM
    result = send(fsm,"FAULT_CLEARED")
    assert result.accepted is True
    assert result.current_state == "FAULT"

    result = send(fsm,"RESET_FAULT")
    assert result.accepted is True
    assert result.current_state =="INIT"

def test_fsm_estop_priority(fsm):
    send(fsm,"INIT_SUCCESS")
    send(fsm,"START_MANUAL")

    result = send(fsm,"ESTOP_TRIGGERED")
    assert result.current_state =="ESTOP"

    # A fault cannot override ESTOP
    result = send(fsm,"FAULT_RAISED")
    assert result.accepted is True
    assert result.current_state == "ESTOP"


    # Releasing ESTOP is not equivalent to reset 
    result = send(fsm,"ESTOP_RELEASED")
    assert result.accepted is True
    assert result.current_state == "ESTOP"

    # Fault is still active 
    send(fsm, "RESET_ESTOP")
    result = send(fsm,"RESET_ESTOP")
    assert result.accepted is False
    assert result.current_state == "ESTOP"

    send(fsm,"FAULT_CLEARED")
    result = send(fsm,"RESET_ESTOP")
    assert result.accepted is True
    assert result.current_state == "INIT"

def test_fsm_unknown_event(fsm):
    result = send(fsm,"INVALID_EVENT")
    assert result.accepted is False
    assert result.previous_state == "INIT"
    assert result.current_state =="INIT"
    assert "Rejected" in result.reason

```

**代码设计说明**

这里有三个重要考虑:

1、每个测试创建独立节点
使用`pytest.fixture` 初始化和销毁`SystemManager`, 避免前一个测试留下的状态影响下一个测试

2、使用真实业务逻辑
测试通过`_handle_send_event()`调用现有代码, 而不是重新写一套模拟FSM

3、