
## 一、整体架构

```
                 ┌────────────────────┐
                 │ GitLab 代码仓库     │
                 │ 172.16.50.250:3000│
                 └─────────┬──────────┘
                           │ 拉代码
                           ▼
                 ┌────────────────────┐
                 │ Jenkins Controller │
                 │ 172.16.50.250:6080│
                 └─────────┬──────────┘
                           │ SSH远程执行
                           ▼
                 ┌────────────────────┐
                 │ Mac Build Agent    │
                 │ 172.16.50.156      │
                 └─────────┬──────────┘
                           │ docker run
                           ▼
                 ┌────────────────────┐
                 │ Android Docker     │
                 │ JDK+SDK+Gradle     │
                 └─────────┬──────────┘
                           │ 编译
                           ▼
                        APK/AAB
```
## 二、角色解释
### 1、GitLab(代码仓库)
地址:
```
 172.168.50.250:3000
```
作用:
```
存放Android 项目源码
```
它负责:
- 管理代码
- 管理分支
- 提交记录
- CI触发(可选)
它不负责:
- 真正编译

### 2、Jenkins Controller(调度中心)

地址:
```
172.16.50:250:6080
```
作用:
```
整个CI/CD的大脑
```
它负责:
- 提供Jenkins Web页面
- 配置Pipeline
- 接收GitLab Webhook
- 调度构建任务
- 管理构建任务
- SSH到其他机器执行任务
它不一定负责:
- 真正编译代码

**因为真正编译的工作在Mac上.**

### 3、Mac Build Agent(编译服务器)
地址: 
```
172.16.50.156
```
作用:
```
真正执行Android编译
```
它负责:
- git clone/pull
- docker run 
- gradle build 
- APK/AAB 生成
为什么需要Mac?

因为未来可能还会打iOS包、签名、Xcode等

### 4、Docker Android Builder(容器编译环境)
运行位置:
```
172.16.50.156 Mac 上
```
作用:
```
隔离Android 编译环境
```
里面包含:
```
JDK
Android SDK
Gradle
build-tools
platform-tools
```
它负责:
```
真正执行./gradlew
```
## 三、真正的执行流程

假设push代码

### 第一步: 代码提交

提交代码到gitlab
```
git push 到 172.16.50.250:3000
```

### 第二步: GitLab通知Jenkins(Webhook)

Gitlab有了新的代码变更就会通知172.16.50.250:6080(Jenkins)

### 第三步:  Jenkins 开始任务

Jenkins 收到了构建任务, 然后SSH到Mac编译服务器
```
ssh robot@172.16.50.156
```

### 第四步: Mac编译服务器更新代码

执行pull指令, 或者reset指令
```
git pull
```


```
git reset --hard origin/main
```
### 第五步: Docker开始编译

```
docker run ...
```
启动Android SDK编译容器

### 第六步: Gradle编译
容器内部执行assembleRelease, 进行打包Apk动作
```
./gradlew assembleRelease 
```
输出: app-release.apk

### 第七步: Jenkins收集结果
Jenkins通过获取APK、保存构建记录、提供下载路径以及通知构建结果给到钉钉


