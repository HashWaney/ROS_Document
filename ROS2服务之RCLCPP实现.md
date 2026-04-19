
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

add_executable(service)
```