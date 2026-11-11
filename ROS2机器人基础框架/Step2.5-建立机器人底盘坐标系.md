目前的ROS 2基础工程已经支持了
-  通过Launch一键启动机器人模型
- 自动加载RViz 2配置
- 正确显示`base_link` 底盘
### 一、创建base_footprint坐标系

1、为什么需要base_footprint?

目前机器人只有`bash_link`, 后续还需要控制机器人移动、计算里程计、接入激光雷达, 因此需要区分两个坐标系:

| | |
|----|




