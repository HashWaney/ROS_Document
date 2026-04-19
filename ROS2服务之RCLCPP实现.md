
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
	auto 

}

```