本次只完成一个功能: 为整机FSM增加故障、急停和受控复位机制, 继续使用现有的`robot_system_manager`, 不创建新Package, 不修改`SendEvent.srv`, 也不接入真实硬件.

### 一、先确定异常状态的行为

![[Pasted image 20261120165859.png]]



| 事件              | 作用                  |
| --------------- | ------------------- |
| FAULT_RAISED    | 触发故障, 进入FAULT       |
| FAULT_CLEARED   | 模拟故障原因已经消除          |
| RESET_FAULT     | 满足条件后, 从FAULT返回INIT |
| ESTOP_TRIGGERED | 触发急停, 进入到ESTOP      |
| ESTOP_RELEASED  |                     |
|                 |                     |