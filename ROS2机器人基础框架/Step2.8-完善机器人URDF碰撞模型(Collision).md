本步目标: 为底盘主体、左右驱动轮和辅助轮万向轮配置碰撞几何, 并通过URDF解析验证.

**一、当前模型与目标**

已有的机器人结构:
```
robot_foundation
└── base_footprint
    └── base_link
        ├── left_wheel_link
        ├── right_wheel_link
        └── caster_mount_link
            └── caster_swivel_link
                └── caster_wheel_link
```

本次需要配置Collision的是四个实体部件:

| Link            | 碰撞几何     | 尺寸              |
| --------------- | -------- | --------------- |
| base_link       | Box      | 0.40x0.30x0.10m |
| left_wheel_link | Cylinder | 半径0.05m、宽0.04m  |
| right_wheel_lin |          |                 |