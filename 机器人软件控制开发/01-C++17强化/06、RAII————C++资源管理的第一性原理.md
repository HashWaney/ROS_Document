
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
锁没有释放,其他线程死锁
内存没释放,长时间运行后崩溃
```
RAII 的核心价值是:
```
不管函数正常返回,还是异常退出,只要对象生命周期结束,析构函数都会被调用.
```
也就是说, 释放逻辑不会被漏掉.

### C++的对象生命周期是确定的

RAII能成立, 是因为C++有一个非常重要的特性:
```
对象什么时候析构, 是确定的.
```
例如:
```
void test(){
	std::string name = "robot";
}// name 在这里立刻析构
```
出了作用域, 局部对象马上析构. 这和很多GC语言不同,GC语言里, 一个对象什么时候真正被回收,通常不是你完全控制的. 而C++可以利用这个确定性,把资源释放放进析构函数里.
所以RAII的第一性原理是:
```
C++对象生命周期是确定的;
资源释放应该绑定到确定的生命周期上;
因此析构函数是释放资源的最佳位置.
```

## How: RAII怎么写?

RAII类通常有一个固定模式.

### 基本结构

```
class ResourceWrapper{
public:
	ResourceWrapper(){
		//获取资源
	}
	~ResourceWrapper(){
		//释放资源
	}
private:
	// 保存资源句柄
}
```
例如文件资源:
```
#include<cstdio>
#include<stdexecpt>

class FileGuard{
public:
	explicit FileGuard(const char* path){
		file_ = std::fopen(path,"w");
		if(file_){
			throw std::runtime_error("failed to open file");
		}
	}
	
	~FileGuard(){
		if(file_){
			std::fclose(file_);
		}
	}

	void write(const char* msg){
		std::fprint(file_, "%s\n", msg);
	}


private:
	FILE* file_ = nullptr;
};
```

使用:

```
void writeRobotLog(){

	FileGuard file("robot.log");
	
	file.write("robot started");
	file.write("motor enabled");
} // 不需要手动调用fclose
``` 

### RAII的关键点: 析构函数一定要安全

析构函数里不要轻易抛出异常.
错误示范:
```
~FileGuard() {
	throw std::runtime_error("close failed");
}
```
为什么不建议?
因为析构函数经常在异常传播过程中被调用. 如果析构函数又抛出异常,可能导致程序直接终止.

正确思路:
```
~FileGuard(){
	if(file_){
		std::fclose(file_);
	}
}
```
如果释放失败, 一般记录日志, 不要抛出异常.

### RAII通常禁止拷贝

很多资源不能随便复制, 比如文件句柄、串口连接、电机连接.
错误情况:
```
FileGuard f1("robot.log");
FileGuard f2 = f1;
```
如果两个对象都认为自己拥有同一个文件句柄, 那么析构时可能会关闭两次.
所以RAII类通常要禁止拷贝:
```
class FileGuard {
public:
	explicit FileGuard(const char* path){
		file_ = std::fopen(path,"w");
		if(!file_){
			throw std::runtime_error("failed to open file");
		}
	}
	~FileGuard(){
		if(file_){
			std::fclose(file_);
		}
	}
	FileGuard(const FileGuard&) = delete;
	FileGuard& operator=(const FileGuard&) = delete;

private:
	FILE* file_;
};
```

这两行很重要, 意思是这个对象不能被复制, 因为它代表一个独占资源.
```
FileGuard(const FileGuard&) = delete;
FileGuard& operator=(const FileGuard&) = delete;
```

### RAII可以支持移动

虽然不能复制, 但有时候可以转移所有权.
比如说: 这个文件句柄原来属于f1, 现在转交给f2; 这就是移动语义;
简单示例:
```
class FileGuard{

public:
	explicit FileGuard(const char* path){
	}

private:

}


```