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

现有维护的分支是: style/refactor2_A10_P_hdmi_ces 和 style/refactor_A10
第一次拉去代码
```
cd ~/jenkins_workspace
git clone http://172.16.50.250:3000/software/hmi_android.git android_app
cd android_app
```
查看远程分支:
```
git branch -r 
```
应该能够看到:
```
origin/style/refactor2_A10_P_hdmi_ces
origin/style/refactory_A10
```
切到HDMI CES分支
```
git checkout -B style/refactor2_A10_P_hdmi_ces origin/style/refactor2_A10_P_hdmi_ces
```
切换到A10分支
```
git checkout -B style/refactor_A10 origin/style/refactor_A10
```
后续Jenkins构建的时候, 传一个分支变量:
```
BRANCH="style/refactor2_A10_P_hdmi_ces"
```
或者是
```
BRANCH="style/refactor_A10"
```
更新代码脚本统一写成:
```
cd ~/jenkins_workspace/android_app
git fetch origin 
git checkout -B "$BRANCH" "origin/$BRANCH"
git reset --hard "origin/$BRANCH"
git clean -fd
```

## 六、配置Git账号权限

如果GitLab需要账号密码, 第一次clone会提示输入
更推荐使用Access Token或SSH Key.
![[Pasted image 20260522162432.png]]

#### #方式一: HTTP+Token
这里的地址取上述截图的地址,不要使用IP地址.

地址类似:
```
git clone -b style/refactor_A10_p_hdmi_ces \
	http://git.dev.utonbot.top:3000/xxx/xxx.git \
	android_app
```

#### #方式二 : SSH Key, 推荐
在Mac编译服务器生成key:
```
ssh-keygen -t rsa -b 4096 -C "jenkins-mac-builder-ci" -f ~/.ssh/id_rsa_jenkins
```
一路回车.
查看公钥:
```
cat ~/.ssh/id_ed25519.pub
```
把输出内容添加到GitLab:
```
GitLab - User setting - SSH Keys 
```
然后测试:
```
➜  .ssh ssh -T git@git.dev.utonbot.top
Welcome to GitLab, @qing.wang1!
```
之后clone项目就可以直接使用:
```
git clone -b style/refactor2_A10_P_hdmi_ces \
	git@git.dev.utonbot.top:222/xxx/xxx.git \
	android_app
```

### 七、测试Android Docker编译环境
进入到项目目录:
```
cd ~/jenkins_workspace
```
先测试Android SDK镜像能不能拉下来:
```
docker run --rm ghcr.io/cirruslabs/android-sdk:35 sdkmanager --version
```
然后测试Gradle:
```
docker run --rm \  
	-v "$PWD":/workspace \  
	-v "$HOME/.gradle":/root/.gradle \  
	-w /workspace \  
	ghcr.io/cirruslabs/android-sdk:35 \  
	bash -c "chmod +x ./gradlew && ./gradlew -v"
```
如果正常, 再正式编译:
```
docker run --rm \  
	-v "$PWD":/workspace \  
	-v "$HOME/.gradle":/root/.gradle \  
	-w /workspace \  
	ghcr.io/cirruslabs/android-sdk:35 \  
	bash -c "chmod +x ./gradlew && ./gradlew clean assembleRelease"
```
APK 一般生成在: app/build/outputs/apk/release/
### 八、编写Jenkins调用脚本
在Mac编译服务器上创建脚本
```
mkdir -p ~/jenkins_scripts
vim ~/jenkins_scripts/build_android.sh
```
写入:
```
#!/bin/bash
set -e 
PROJECT_DIR = "/Users/robot/jenkins_workspace/hmi"

# 从第一个参数读取分支名
BRANCH="$1"
if [ -z "$BRANCH" ]; then
	echo "错误: 请传入需要构建的分支名"
	echo "示例"
	echo "./build_android.sh style/refactor_A10"
	echo "./build_android.sh style/refactor2_A10_P_hdmi_ces"
	exit 1
fi

cd "$PROJECT_DIR"

echo "==========当前构建分支: $BRANCH ==============="

git fetch origin 
git checkout -B "$BRANCH" "origin/$BRANCH"
git reset --hard "origin/$BRANCH"
git clean -fd 

docker run --rm \
	-v "$PROJECT_DIR":/workspace \
	-v "$HOME/.gradle":/root/.gradle \
	-w /workspace \
	ghcr.io/cirruslabs/android-sdk:35 \
	bash -c "chmod +x ./gradlew && ./gradlew clean assemableRelease"

ls -lh "$PROJECT_DIR/app/build/outputs/apk/release/"

```
保存后赋予执行权限
```
chmod +x ~/jenkins_scripts/build_android.sh
```
手动测试, 执行切换分支的时候
```
~/jenkins_scripts/build_android.sh style/refactor_A10
```

### 九、Jenkins里远程调用这个脚本

Jenkins Controller部署在172.16.50.250服务器里面, 那么Jenkins任务通过SSH执行Mac编译服务器上的脚本:

```
ssh robot@172.16.50.156 "/Users/robot/jenkins_scripts/build_android.sh"
```
也就是说Jenkins 不需要懂Android SDK, 不需要安装Gradle, 不需要安装Docker; 
只需要负责:
```
SSH 到 Mac -> 执行 build_android.sh
```

### 十、最终关系总结
```
Jenkins Controller : 172.16.50.250
	 | SSH
Mac 编译服务器: 172.16.50.156
	 | git fetch / checkout
GitLab: 172.16.50.250:3000
	 | docker run
Android SDK 容器
	 | ./gradlew assemableRelease
生成APK
```
