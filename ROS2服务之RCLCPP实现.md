
## 创建功能包和节点

```
cd chapt3/chapt3_ws/src
ros2 pkg create example_service_rclcpp --build-type ament_cmake --dependencies rclcpp
touch example_service_rclcpp/src/service_server_01.cpp
touch example_service_rclcpp/src/service_client_01.cpp
```

编写服务端节点:
```
#include "rclcpp/rclcpp.hpp"

class ServiceServer01 : public rclcpp::Node{

public:
	ServiceServer01(std::string name): Node(name){
		RCLCPP_INFO(this->get_logger(),"节点已经启动 %s.",name.c_str());
	}
private:
};
int main(int argc, char** argv){
	rclcpp::init(argc,argv);
	auto node = std::make_shared<ServiceServer01>("service_server_01");
	rclcpp::spin(node);
	rclcpp::shutdown();
	return 0;
}

```

编写客户端节点:

```
#include "rclcpp/rclcpp.hpp"
class ServiceClient01 : public rclcpp::Node{

public:
	ServiceClient01(std::string name) : Node(name){
		RCLCPP_INFO(this->get_logger(),"节点已经启动%s.",name.c_str());
		
	}
private:
}

int main(int argc, char** argv){
	rclcpp::init(argc,argv);
	auto node = std::make_shared<ServiceClient01>("service_client_01");
	rclcpp::spin(node);
	rclcpp::shutdown();
	return 0;
}

```
CMakeLists.txt
```
add_executable(service_client_01 src/service_client_01.cpp)
ament_target_dependencies(service_client_01 rclcpp)

add_executable(service_service_01 src/service_server_01.cpp)
ament_target_dependencies(service_server_01 rclcpp)

install(
	TARGETS
	service_server_01 
	service_client_01
	DESTINATION lib/${PROJECT_NAME}
)
```

编译:
```
colcon build -packages-select example_service_rclcpp

source install/setup.bash
ros2 run example_service_rclcpp service_server_01

#新终端

source install/setup.bash
ros2 run example_service_rclcpp service_client_01
```
# 服务端实现

## 导入接口
两数相加,需要利用ROS2自带的example_interfaces接口, 使用命令行可以查看这个接口的定义.
```
ros2 interfaces show example_interfaces/srv/AddTwoInts
```

![[Pasted image 20260419155721.png]] 
导入接口的三个步骤:
1、在CMakeLists.txt中导入, 具体是先find_packages再ament_target_dependencies
2、在packages.xml中导入, 具体是添加depend标签并将消息接口写入.
3、在代码中导入,C++中是#include ”消息功能包/xxxx/xxx.hpp“.

更改后的CMakeLists.txt文件:
```
find_package(example_interfaces REQUIRED)


add_executable(service_client_01 src/service_client_01.cpp)
ament_target_dependencies(service_client_01 rclcpp example_interfaces)

add_executable(service_server_01 src/service_server_01.cpp)
ament_target_dependencies(service_server_01 rclcpp example_interfaces)
```

package.xml文件
```
<depend>example_interfaces</depend>
```
导包:
```
#include "example_interfaces/srv/add_two_ints.hpp"
```

## 编写代码
```
#include "example_interfaces/srv/add_two_ints.hpp"
#include "rclcpp/rclcpp.hpp"

class ServiceServer01 : public rclcpp::Node{

public:
	ServiceServer01(std::string name) : Node(name){
		RCLCPP_INFO(this->get_logger(),"节点已经启动 %s.“,name.c_str());
		
		//创建服务
		
	
	}


private:
	// 声明一个服务
	rclcpp::Service<example_interfaces::srv::AddTwoInts>::SharedPtr add_ints_server_;
	
	void handle_add_two_ints(
		const std::shared_ptr<example_interfaces::srv::AddTwoInts::Request> request,std::shared_ptr<example_interfaces::srv::AddTwoInts::Response> response
	){
		RCLCPP_INFO(this->get_logger(),"recv a: %ld, b: %ld",request->a, request->b);
		response->sum = request->a + request->b;
	};

};



```