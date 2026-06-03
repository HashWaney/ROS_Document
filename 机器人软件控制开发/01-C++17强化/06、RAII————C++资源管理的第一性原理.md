
## 核心结论

RAII的本质不是一个语法点, 而是一种思想
```
把资源的申请和释放,绑定在对象的生命周期上.对象创建时获取资源,对象销毁时候释放资源.
```
也就是说, 在C++里面, 不要到处手动写:
```
open();
close();
```
而是应该让对象自己负责:
```
构造函数里面 open
析构函数里面 close
```
这样的代码更安全, 也更适合复杂工程.

## What: RAII 是什么?

RAII 全称: Resource Acquisition Is Initialization (资源获取即初始化), 这个翻译有点抽象:

```
只要一个对象成功创建,它就已经拿到了某种资源.
只要这个对象生命周期结束,它就自动释放这个资源.
```
这里的资源不只是内存.

在C++工程中, 资源可以是:
```
内存
文件句柄
网络连接
串口连接
蓝牙连接
互斥锁
线程
数据库连接
GPU显存
电机控制句柄
传感器连接
日志文件
```

比如一个机器人程序要连接电机:
```
连接电机 = 获取资源
断开电机 = 释放资源
```
普通写法可能是

```
Motor motor;
motor.connect();
motor.sendCommand(1.0);
motor.disconnect();
```

问题是: 如果中间出错了?
```
Motor motor;
motor.connect();

throw std::runtime_error("controll err");

motor.disconnect(); // 这一行不会执行


```
这样电机连接可能没有释放.

RAII的写法就是:
```
class MotorConnection {

public: 
	MotorConnection(){
		connect();
	}

	~MotorConnection(){
		disconnect();
	}
	void sendCommand(double velocity)
	{
	}

private:
	void connect(){
		// 打开电机连接
	}
	void disconnect(){
		// 关闭电机连接
	}
};

```

使用时:

```
void runRob
```