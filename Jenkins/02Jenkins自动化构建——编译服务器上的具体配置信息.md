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
`````
