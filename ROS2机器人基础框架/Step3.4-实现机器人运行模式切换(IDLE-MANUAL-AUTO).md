
本次只实现一个功能: 允许机器人在IDLE、MANUAL、AUTO三种状态之间安全切换


### 一、设计状态转换规则

![[Pasted image 20261120152613.png]]

| 当前状态   | 事件            | 目标状态   | 是否允许 |
| ------ | ------------- | ------ | ---- |
| IDLE   | START_MANUAL  | MANUAL | 是    |
| MANUAL | STOP          | IDLE   | 是    |
| IDLE   | START_AUTO    | AUTO   | 是    |
| AUTO   | STOP          | IDLE   | 是    |
| AUTO   | TASK_FINISHED | IDLE   | 是    |
| MANUAL | START_AUTO    | 不变     | 否    |
| AUTO   | START_MANUAL  | 不变     | 否    |

这里有一条重要原则: 手动模式和自动模式不能直接相互