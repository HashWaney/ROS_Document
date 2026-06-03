
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
void runRobot(){
	MotorConnection motor;
	
	motor.sendCommand(1.0);
}
// runRobot 结束后,motor自动析构, disconnect 自动执行
```

##  Why: 为什么C++需要RAII?

这是重点, 从第一性原理理解.
### C++和Java/Kotlin最大区别之一: 没有自动GC管理所有资源
Java、Kotlin、Dart里有垃圾回收机制, 内存对象不用手动delete, 但是C++不一样.
C++追求的是:
```
性能可控
内存可控
生命周期可控
接近硬件
```
所以C++不会默认帮你做所有资源回收. 而是机器人控制恰恰很看重这些:
```
什么时候分配内存
什么时候释放内存
什么时候打开设备
什么时候关闭设备
控制线程能不能稳定运行
是否因为GC暂停导致控制卡顿
```
如果机器人控制循环跑在1kHz, 也就是1ms一次, 如果突然有不可控的GC暂停,就可能导致控制不稳定, 所以C++更适合这类底层系统, 但代价是:
```
资源管理必须非常严谨
```
RAII就是C++用来解决资源管理问题的核心机制.

### 手动释放资源容易出错

假设你手动管理文件:
```
FILE* file = fopen("robot.log","w");
fprint(file,"robot started\n);
fclose(file);
```
看起来没问题. 
但是真实代码会有各种分支, 如下:
```
FILE* file = fopen("robot.log","w");

if(!file){
	return;
}

fprint(file,"robot started\n");

if(someError){
	return; // fclose 没有执行.
}
fclose(file);

```
这里就出现了资源泄露. 
在机器人软件里, 如果类似问题发生在:
```
电机连接
串口连接
相机句柄
雷达驱动
GPU buffer
控制线程
```
后果会更严重.
比如说: 
```
串口没关闭, 下次连接失败
线程没停止, 程序退出卡死

```