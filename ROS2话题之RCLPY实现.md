

# 1、创建功能包和节点

创建功能包
```
ros2 pkg create example_topic_rclpy --build-type ament_python --dependencies rclpy
```

创建节点文件
```
cd example_topic_rclpy
touch topic_subscribe_02.py
touch topic_publisher_02.py
```