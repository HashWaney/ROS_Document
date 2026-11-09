
### 2.1 确定机器人模型的坐标系
先以一个通用的移动机器人作为开发模型,未来再根据真实底盘修改尺寸和结构.

1、设计TF坐标树
 ![[Pasted image 20261109171352.png]]
 坐标约定遵循ROS REP-103 和 REP-105:
 - X轴: 机器人前方
 - Y轴: 机器人左方
 - Z轴: 机器人上方
map -> odom 以后由定位或SLAM算法发布; odom->base_footprint由里程计系统发布. 这两段动态关系不写成URDF的固定关节.
对于第一版URDF, 只负责从base_footprint开始描述机器人本体、轮子和雷达的结构

2、本次执行: 创建URDF目录
在Ubuntu中执行:
```
cd ~/robot_foundation_ws/src/robot_description

mkdir -p urdf

ls -al urdf
```
暂时不创建XML文件, 因为还需要确定底盘模型

3、确定第一个机器人模型
考虑到后续要加入运动节点,故障节点和2D雷达, 采用双轮差速移动底盘作为基础模型,尺寸全部保持待定.


### 2.2 创建机器人的第一个Link -base_link
本次只完成一件事: 使用Xacro定义机器人底盘主体base_link, 并验证能否正确生成URDF

1、理解base_link
base_link 是机器人的本体参考坐标系. 后续左右驱动轮、辅助万向轮和激光雷达都将以它为参考建立连接关系.
![[Pasted image 20261109174654.png]]

先赢一个长方体表示底盘、以下尺寸仅用于开发验证, 并非真实机器人尺寸


