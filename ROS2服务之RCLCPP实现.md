
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
		add_ints_server_ = this->create_service<example_interfaces::srv::AddTwoInts>("add_two_ints_srv", std::bind(&ServiceServer01::handle_add_two_ints,this,std::placeholders::_1,std::placeholders::_2));
	
	}


private:
	// 声明一个服务
	rclcpp::Service<example_interfaces::srv::AddTwoInts>::SharedPtr add_ints_server_;
	// 收到请求的处理函数
	void handle_add_two_ints(
		const std::shared_ptr<example_interfaces::srv::AddTwoInts::Request> request,std::shared_ptr<example_interfaces::srv::AddTwoInts::Response> response
	){
		RCLCPP_INFO(this->get_logger(),"recv a: %ld, b: %ld",request->a, request->b);
		response->sum = request->a + request->b;
	};

};
int main(int argc, char** argv){
	rclcpp::int(argc,argv);
	auto node = std::make_shared<ServiceServer01>("service_server_0");
	rclcpp::spin(node);
	rclcpp::shutdown();
	return 0;
}
```


# 客户端实现

## API 接口

### create_client

参数: service_name , qos_profile, group
### async_send_request
参数: request , CallBack

### wait_for_service
参数: 等待时间, 返回值是bool类型, true 则为上线, false为不上线.

## 编写代码
```
#include "example_interfaces/srv/add_two_ints.hpp"

class ServiceClient01 : public rclcpp::Node{

public:
	ServiceClient01(std::string name):Node(name){
		RCLCPP_INFO(this->get_logger(),"节点启动: %s.",name.c_str());
		// 创建客户端
		client_ = this->create_client<example_interfaces::srv::AddTwoInts>(
		"add_two_ints_srv");
		
	}
	void send_request(int a,int b){
		RCLCPP_INFO(this->get_logger(),"calc %d+%d",a,b);
		while(!client_->wait_for_service(std::chrono::seconds(1))){
			if(!rclcpp::ok()){
				RCLCPP_ERROR(this->get_logger(),"interrupt service");
				return;
			}
			RCLCPP_INFO(this->get_logger(),"waitint for service");
		}
		// 请求
		auto request = std::make_shared<example_interfaces::srv::AddTwoInts_Request>();
		request->a =a;
		request->b =b;
		// send request 
		client_->async_send_request(
			request, std::bind(&ServiceClient01::result_callback_,this,std::placeholders::_1)
		);
	
	};
	

private:
	// 声明客户端
	rclcpp::Client<example_interfaces::srv::AddTwoInts>::SharedPtr client_;
	void result_callback_(
		rclcpp::Client<example_interfaces::srv::AddTwoInts>::SharedFuture
		result_future
	){
	
		auto response = result_future.get();
		RCLCPP_INFO(this->get_logger(),"calc result %ld",response->sum);
	}

};
int main(int argc, char** argv){
	rclcpp::init(argc, argv);
	auto node =std::make_shared<ServiceClient01>("service_client_01");
	node->send_request(5,6);
	rclcpp::spin(node);
	rclcpp::shutdown();
	return 0;

}

```

开启两个zho g n