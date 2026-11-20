本次只完成一个功能: 为整机FSM增加故障、急停和受控复位机制, 继续使用现有的`robot_system_manager`, 不创建新Package, 不修改`SendEvent.srv`, 也不接入真实硬件.

### 一、先确定异常状态的行为

