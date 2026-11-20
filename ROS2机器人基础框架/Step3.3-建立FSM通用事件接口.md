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
