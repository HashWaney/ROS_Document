# 服务通信介绍

服务分为客户端和服务端, 例如平时手机APP可以认定为是客户端, APP服务器对于软件来说就是服务端.
客户端发送请求给服务端, 服务端可以根据客户端的请求做一些处理, 然后返回结果给客户端.
![[Pasted image 20260419132831.png|371]]
所以服务-客户端模型, 也可以理解为请求-响应模型.

那么服务端和话题的不同之处在于: 话题是没有返回的, 适用于单向或大量的数据传递. 而服务是双向的, 客户端发送请求, 服务端响应请求.

同时服务还有一些注意事项:
- 同一个服务(名称相同)有且只能有一个节点提供
- 同一个服务可以被多个客户端调用.



![[Service-SingleServiceClient.gif]]![[Service-MultipleServiceClient.gif]]
# 服务初体验

## 启动服务端
打开终端, 运行下面命令, 这个命令用于运行一个服务节点, 这个服务的功能是将两个数字相加,给定a,b两个数, 返回sum也就是a和b之和.
```
ros2 run example_rclpy_minimal_service service
```
## 使用命令查看服务列表
```
ros2 service list
```

![[Pasted image 20260419134608.png]]

## 手动调用服务
再启动一个终端, 输入下面的命令(注: a: 、b:后的空格)
```
ros2 service call /add_two_ints example_interfaces/srv/AddTwoInts "{a:5,b:10}"
```

![[Pasted image 20260419135116.png]]


# ROS2服务常用命令
## 查看服务列表
```
ros2 service list
```

![[Pasted image 20260419135213.png]]

## 手动调用服务

```
ros2 service call /add_two_ints example_interfaces/srv/AddTwoInts "{a: 4, b: 10}"
```


## 查看服务接口类型

```
ros2 service type /add_two_ints
```
