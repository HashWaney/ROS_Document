
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
import rclcpy





```