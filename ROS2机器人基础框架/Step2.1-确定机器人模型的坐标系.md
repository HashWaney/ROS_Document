
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

