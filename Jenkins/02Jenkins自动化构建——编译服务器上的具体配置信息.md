## 目标

Mac 编译服务器
├─ 安装 Git
├─ 安装 Docker / Colima
├─ 拉取 Android 工程
├─ 固定使用分支 style/refact_A10_P_ces
├─ 使用 Docker 编译 Android APK
└─ 给 Jenkins 提供可执行脚本

## 一、SSH登录Mac编译服务器

在你的电脑上或Jenkins服务器上执行
```
ssh robot@172.16.50.156
```
进入后确认当前用户:
```
whoami 
pwd
```
应该能看到
```
robot
/Users/robot
```
## 二、安装基础工具

### 1、检查Homebrew

```
brew --version
```
如果没有安装Homebrew, 需要先安装:
```
➜  ~ brew --version
Homebrew 5.0.5
➜  ~
```

### 2、安装Git
```
brew install git 
```
验证:
```
git --version
```
```
➜  ~ git --version
git version 2.39.5 (Apple Git-154)
```
## 三、安装Docker环境

Mac编译服务器推荐使用Colima+Docker CLI
### 1、安装Docker和Colima

```
brew install docker docker-compose colima 
```
### 2、启动Colima
```
colima start 
```
如果机器资源够用, 可以指定CPU和内存:
```
colima start --cpu 4 --memory 8 --disk 80
```
### 3、验证Docker
```
docker --version
docker ps 
```
再执行:
```
docker run --rm hello-world
```
看到输出Hello from Docker !
说明Docker正常

## 四、创建Jenkins工作目录

建议统一放到/Users/root/jenkins_workspace

执行: 
```
mkdir -p ~/jenkins_workspace

cd ~/jenkins_workspace
```
## 五、拉取Android工程代码

现有维护的分支是: 