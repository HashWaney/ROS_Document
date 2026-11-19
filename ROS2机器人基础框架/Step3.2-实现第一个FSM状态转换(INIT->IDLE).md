
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
zhe