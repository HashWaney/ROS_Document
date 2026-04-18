

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
编写发布者代码

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

编写发布者代码
```
#!/usr/bin/env python3
from rclpy.node import Node
from std_msgs.msg import String
class NodeSubscribe02(Node):
	def __init__(self,name):
		super().__init__(name)
		self.get_logger().info("hello i am %s" %name)
		#创建订阅者
		self.command_subscribe_ = self.create_subscription(String,"command",self.command_callback,10)
	def command_callback(self,msg):
		speed = 0.0
		if msg.data == "backup":
			speed = 0.2
		self.get_logger().info(f'收到{msg.data}命令,发送速度{speed}')

def main(args=Node):
	rclpy.init(args=args)
	node = NodeSubscribe02("topic_subscribe_02")
	rclpy.spin(node)
	rclpy.shutdown()
		
```

# 运行测试
## 发布节点
```
source install/setup.bash
ros2 run example_topic_rclpy topic_publisher_02
```
## 订阅节点
```
source install/setup.bash
ros2 run example_topic_rclpy topic_subscribe_02
```