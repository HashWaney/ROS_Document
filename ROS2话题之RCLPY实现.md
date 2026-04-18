

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
编写发布者和订阅节点代码

```
#!/usr/bin/env python3 
import rclpy
from rclpy.node import Node
from std_msgs.msg import String

class NodePublisher02(Node):
	def __init__(self,name):
		super().__init__(name)
		self.get_logger().info("hello i am %s" % name)
		# 创建发布者
		self.command_publisher_ = self.create_publisher(String,"command",10)
		self.timer =  self.create_timer(0.5,self.timer_callback)
	def timer_callback(self):
		"""
			定时回调函数
		"""
		msg = String()
		msg.data = "backup1"
		self.command_publisher_.publish(msg)
		self.get_logger().info(f'发布了指令:{msg.data}')
		
def main(args=None):
	rclpy.init(args=args)
	node = NodePublisher02("topic_publisher_02")
	rclpy.spin(node)
	rclpy.shutdown()

```


```
#!

```