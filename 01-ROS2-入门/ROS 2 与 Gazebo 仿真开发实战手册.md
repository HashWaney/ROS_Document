## 一、 项目创建详细步骤 (从零开始)

### 1. 环境准备

确保已安装 ROS 2 Jazzy 和 Gazebo Sim。

### 2. 创建工作空间 (Workspace)

Bash

```
mkdir -p ~/Desktop/Cpp/ros2_ws/src
cd ~/Desktop/Cpp/ros2_ws/src
```

### 3. 创建功能包 (Package)

Bash

```
ros2 pkg create --build-type ament_cmake ros2_demo_node --dependencies rclcpp geometry_msgs
```

### 4. 编写 C++ 节点

在 `src/ros2_demo_node/src/` 下创建 `my_node.cpp`：

C++

```
#include "rclcpp/rclcpp.hpp"
#include "geometry_msgs/msg/twist.hpp"

class MoveNode : public rclcpp::Node {
public:
    MoveNode() : Node("gazebo_move_node") {
        publisher_ = this->create_publisher<geometry_msgs::msg::Twist>("/cmd_vel", 10);
        timer_ = this->create_wall_timer(std::chrono::milliseconds(100), std::bind(&MoveNode::timer_callback, this));
    }
private:
    void timer_callback() {
        auto msg = geometry_msgs::msg::Twist();
        msg.linear.x = 0.5;  // 前进速度
        msg.angular.z = 0.5; // 旋转速度
        publisher_->publish(msg);
    }
    rclcpp::Publisher<geometry_msgs::msg::Twist>::SharedPtr publisher_;
    rclcpp::TimerBase::SharedPtr timer_;
};

int main(int argc, char **argv) {
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<MoveNode>());
    rclcpp::shutdown();
    return 0;
}
```

### 5. 配置 CMakeLists.txt

在文件末尾添加：

CMake

```
add_executable(demo_exe src/my_node.cpp)
ament_target_dependencies(demo_exe rclcpp geometry_msgs)

# 安装可执行文件到 lib
install(TARGETS demo_exe DESTINATION lib/${PROJECT_NAME})

# 安装 launch 文件夹到 share
install(DIRECTORY launch DESTINATION share/${PROJECT_NAME})
```

### 6. 创建 Launch 文件

在 `src/ros2_demo_node/launch/` 下创建 `demo_launch.py` (**方案 B：最优解**):

Python

```
from launch import LaunchDescription
from launch_ros.actions import Node

def generate_launch_description():
    return LaunchDescription([
        # 1. 桥接器 (直接对接 Gazebo 全名)
        Node(
            package='ros_gz_bridge',
            executable='parameter_bridge',
            arguments=['/model/vehicle_blue/cmd_vel@geometry_msgs/msg/Twist@gz.msgs.Twist'],
            output='screen'
        ),
        # 2. C++ 节点 (通过重映射将 /cmd_vel 指向小车真名)
        Node(
            package='ros2_demo_node',
            executable='demo_exe',
            output='screen',
            remappings=[('/cmd_vel', '/model/vehicle_blue/cmd_vel')]
        )
    ])
```

---

## 二、 常见问题及解决方案 (FAQ)

### 1. 编译报错：CMakeCache.txt 路径不匹配

- **现象**：提示路径从 `C++` 变成了 `Cpp`。
    
- **原因**：CMake 锁死了绝对路径，移动或重命名文件夹后缓存失效。
    
- **解决**：彻底删除 `build/`, `install/`, `log/` 文件夹并重新编译。
    

### 2. 代码修改后不生效

- **原因**：ROS 2 运行的是 `install/` 下的副本。
    
- **解决**：
    
    - 使用 `colcon build --symlink-install`（软链接模式）。
        
    - 修改 C++ 后必须点击 CLion 的 **Build (小锤子)** 或重新执行 `colcon build`。
        
    - 重启 Launch 终端并重新 `source install/setup.bash`。
        

### 3. Gazebo Reset (重置) 后小车不动

- **现象**：`ros2 topic echo` 有数据，但小车不走。
    
- **排查步序**：
    
    1. **检查播放状态**：Reset 后 Gazebo 常会自动暂停，需手动点击左下角 **Play**。
        
    2. **检查时钟**：确认底部的 `Sim Time` 在跳动。
        
    3. **检查桥接链路**：Reset 会导致 Gazebo 话题重建。此时应 **先开 Gazebo Play -> 再启动 Launch/Bridge**。
        
    4. **物理死锁**：若 `gz topic -e` 有数据但车不动，可能是模型陷进地板，用鼠标提一下车。
        

### 4. 桥接器 (Bridge) 找不到数据

- **原因**：重映射路径配置错误。
    
- **解决方案 (方案 B)**：
    
    - 让 Bridge 直接搬运 Gazebo 原生话题（如 `/model/vehicle_blue/cmd_vel`）。
        
    - 在 Launch 文件中利用 `remappings` 让你的节点去适配这个长名字。
        

---

## 三、 核心调试指令速查表

| **目的**           | **指令**                                           |
| ---------------- | ------------------------------------------------ |
| **彻底清理**         | `rm -rf build/ install/ log/`                    |
| **编译项目**         | `colcon build --symlink-install`                 |
| **查看 ROS 话题**    | `ros2 topic list`                                |
| **查看 Gazebo 话题** | `gz topic -l`                                    |
| **监听 ROS 数据**    | `ros2 topic echo /cmd_vel`                       |
| **监听 Gazebo 数据** | `gz topic -e -t /model/vehicle_blue/cmd_vel`     |
| **手动发包测试**       | `ros2 topic pub /model/vehicle_blue/cmd_vel ...` |
| **启动仿真(自动播放)**   | `gz sim -r diff_drive.sdf`                       |
## 四、 开发最佳实践总结

1. **启动顺序**：先启动 Gazebo -> 点击 Play -> 启动 Launch。
    
2. **路径管理**：避免文件夹名使用 `C++` 等特殊字符。
    
3. **名字管理 (方案 B)**：C++ 代码写短名字（如 `/cmd_vel`），Launch 文件负责将其“重映射”到具体的机器人（如 `/model/vehicle_blue/cmd_vel`）。这样代码最稳健，也最容易扩展到多机器人系统

## 五、 实战问题排查与终极解决方案 (Troubleshooting)

### 1. 路径中特殊字符导致的编译崩溃

- **遇到的问题**：执行 `colcon build` 时，CMake 报出莫名其妙的路径错误，或者找不到头文件。
    
- **根本原因**：你的工作空间路径中包含了 `C++` 这种特殊字符（如 `~/Desktop/C++/ros2_ws`）。CMake 在处理带 `+` 号的路径时会产生解析歧义。
    
- **解决方案**：将文件夹重命名为不带特殊符号的名称，例如 `Cpp_Project`。
    
- **教训**：**ROS 2 开发中，所有层级的文件夹名、包名、文件名应严格遵守：仅限小写字母、数字和下划线。**
    

### 2. Launch 文件启动后找不到节点 (Package not found)

- **遇到的问题**：运行 `ros2 launch ...` 提示 `Package 'ros2_demo_node' not found`。
    
- **原因分析**：
    
    1. 没有执行 `source install/setup.bash`。
        
    2. `CMakeLists.txt` 中漏写了 `install(DIRECTORY launch ...)`，导致 Launch 文件没被拷贝到 install 目录。
        
- **解决方案**：
    
    - 检查 `CMakeLists.txt` 是否有安装指令。
        
    - 执行 `colcon build` 后，去 `install/ros2_demo_node/share/` 下看看有没有 `launch` 文件夹。
        

### 3. Gazebo 仿真环境“假死”与 Reset 逻辑冲突

- **遇到的问题**：点击 Gazebo 里的 Reset 按钮后，ROS 话题依然在发数据，但小车纹丝不动。
    
- **深度解析**：
    
    - Gazebo 的 **Reset Model/World** 会重置仿真时钟和物理状态。
        
    - 如果此时 `ros_gz_bridge` 没有检测到话题重建，连接就会断开。
        
    - **最隐蔽的坑**：Jazzy 版 Gazebo 在重置后，有时会自动进入 **Pause** 状态。
        
- **解决方案**：
    
    1. **观察时钟**：看 Gazebo 底部状态栏的 `Sim Time` 是否在走动。
        
    2. **重连三部曲**：先点 Play -> 终端 Ctrl+C 停止 Launch -> 重新运行 Launch。
        

### 4. 话题重映射（Remapping）失效

- **遇到的问题**：`ros2 topic list` 能看到 `/cmd_vel`，但小车没反应。
    
- **原因分析**：你在 C++ 代码里写的是 `/cmd_vel`，但 Gazebo 期望的是 `/model/vehicle_blue/cmd_vel`。如果你在 `ros_gz_bridge` 上做了重映射，但没在控制节点上做，数据流就接不上。
    
- **解决方案**：
    
    - **原则**：代码（Node）发什么，桥接器（Bridge）就转什么。
        
    - 在 Launch 中，给控制节点加上 `remappings=[('/cmd_vel', '/model/vehicle_blue/cmd_vel')]`。
        
    - **调试技巧**：使用 `ros2 topic info /model/vehicle_blue/cmd_vel` 查看当前的 `Publisher` 和 `Subscription` 数量，如果 `Subscription` 为 0，说明数据没进 Gazebo。
        

### 5. `colcon build` 修改不生效

- **遇到的问题**：改了 Launch 文件里的参数，重新运行发现还是旧的结果。
    
- **原因分析**：没有使用符号链接安装，系统一直在运行 `install/` 目录下的旧备份。
    
- **解决方案**：
    
    - 必须使用 `colcon build --symlink-install`。
        
    - 这样 `install/` 文件夹里的文件直接链接到 `src/` 里的源文件。**改完 Python/Launch 脚本后，直接运行即可，无需再次编译。**
        

---

## 六、 常用维护脚本建议

为了方便你后续开发，建议在 `ros2_ws` 根目录下创建一个 `clean_build.sh`，用于解决大部分奇葩编译问题：

Bash

```
#!/bin/bash
# 清理并重新编译脚本
rm -rf build/ install/ log/
colcon build --symlink-install
source install/setup.bash
echo "工作空间清理并编译完成！"
```