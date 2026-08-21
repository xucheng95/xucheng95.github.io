---
title: ZERO 机械臂学习之路 07：上电与复位（限位开关 / 硬复位 / 软复位）
date: 2026-08-19 21:50:00
tags:
- 机械臂
- 限位开关
- 复位
- 联调
categories: ZERO机械臂
mathjax: false
---

> 这是「ZERO 机械臂学习之路」系列的第 07 篇。机械臂有个"失忆症"：断电后它不知道自己转到哪了。这一篇讲它怎么找回自己：**限位开关**（触觉）、**硬复位**（撞开关找零点）、**软复位**（不撞开关直接回位）。这是上电联调的第一步。

## 1. 为什么需要复位

步进电机的编码器记录的是"相对位置"。断电再上电，**主控不知道电机现在转到哪个角度**——如果直接发命令让它动，它可能朝着错误方向乱转，甚至撞坏结构件。

所以开机第一件事：**让机械臂回到一个已知的位置（零点）**。ZERO 提供了两条路：

| 方式 | 命令 | 原理 | 需要限位开关 |
|------|------|------|:---:|
| 硬复位 | `hard_reset` | 每个关节转到撞上限位开关为止，把撞到的位置记为 0° | ✅ |
| 软复位 | `soft_reset` | 不撞开关，按当前角度走最短路径回到初始化角度 | ❌ |

## 2. 限位开关：机械臂的"触觉"

### 硬件

BOM 里有 **5 个限位开关**（KW11-3Z-A ×3 + KW11-3Z-B ×2，A/B 只是杠杆形状不同），接在 STM32 的 **PD0~PD5**（`JOINT_LIMIT_1_Pin` ~ `JOINT_LIMIT_6_Pin`）。

> 注意：**6 个关节只有 5 个限位开关**——关节 6（末端旋转）没有装。代码里也印证了这一点（下面硬复位部分会看到）。

限位开关是个**微动开关**：关节转到底碰到杠杆 → 开关接通/断开 → GPIO 电平变化 → 触发中断。

### 代码：中断链路

限位开关接的是 **EXTI 外部中断**（`stm32f4xx_it.c`）：

```c
// EXTI0_IRQHandler ~ EXTI5_IRQHandler（PD0~PD5 各占一条中断线）
void EXTI0_IRQHandler(void) {
    HAL_GPIO_EXTI_IRQHandler(JOINT_LIMIT_1_Pin);
    ...
}

void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    uint32_t joint_id = robot_joint_pin2id(GPIO_Pin);   // 引脚 → 关节号
    robot_joint_limit_happend(joint_id);                 // 记录限位触发 + 发事件
    __HAL_GPIO_EXTI_CLEAR_IT(GPIO_Pin);                  // 清中断标志（防抖动误触发）
}
```

`robot_joint_limit_happend` 会把 `ROBOT_STATUS_LIMIT_HAPPENED` 状态位置位，并往事件队列塞一个 `ROBOT_LIMIT_SWITCH_EVENT`——控制任务收到后做后处理（清状态位、延时 200ms 等开关稳定）。

**这是典型的"中断只做最轻的事，重活交给任务"**：中断里只置位 + 发事件，200ms 的抖动等待放在任务里。

## 3. 硬复位：撞限位开关找零点

`hard_reset` 命令 → `robot_joint_hard_reset()`（`robot.c`）：

```c
static void robot_joint_hard_reset(void)
{
    // todo: 后面不用-2，直接从ROBOT_MAX_JOINT_NUM - 1开始
    for (int i = ROBOT_MAX_JOINT_NUM - 2; i >= 0; i--) {   // 关节5 → 关节1
        ROBOT_STATUS_SET(g_robot.joints[i].status, ROBOT_STATUS_LIMIT_ENABLE);
        robot_joint_reset(i);
        vTaskDelay(100);
    }
    // 复位完成后：把所有关节角度置为初始化值，末端位置清零
    for (int i = 0; i < ROBOT_MAX_JOINT_NUM; i++) {
        g_robot.joints[i].current_angle = g_joints_init[i].current_angle;
    }
    g_robot.cur_pos.x = g_robot.cur_pos.y = g_robot.cur_pos.z = 0;
}
```

两个细节值得注意：

1. **循环从 `ROBOT_MAX_JOINT_NUM - 2`（关节5）开始，到关节1 结束**——关节 6 不在硬复位范围里（它没有限位开关）。代码注释里作者自己留了 TODO："后面不用 -2，直接从 `ROBOT_MAX_JOINT_NUM - 1` 开始"（意思是等装了关节 6 的限位开关后改掉）。
2. **先复位的关节先动**：从靠近末端的关节 5 往底座方向逐个复位，避免还没复位的关节被运动中的关节带跑。

每个关节的复位逻辑 `robot_joint_reset(joint_id)`：

```c
// 1. 先读一次限位开关：如果已经触发，直接清零编码器位置，完事
state = robot_get_limit_status(joint_id);
if (state == GPIO_PIN_SET) {
    Emm_V5_Reset_CurPos_To_Zero(joint_id + 1);   // 张大头驱动器：当前位置清零
    return;
}

// 2. 没触发 → 按复位方向转 360°（默认角度），边转边等中断
robot_joint_rotate_to(joint_id, reset_dir, ROBOT_RESET_DEFAULT_ANGLE, ...);
while (!ROBOT_STATUS_IS(g_robot.joints[joint_id].status, ROBOT_STATUS_LIMIT_HAPPENED)) {
    vTaskDelay(200);   // 200ms 轮询一次限位状态
}
Emm_V5_Reset_CurPos_To_Zero(joint_id + 1);   // 撞到开关 → 此位置记为 0°
```

**复位方向**来自关节配置表里的 `reset_dir` 字段（`DIR_POSITIVE` / `DIR_NEGATIVE`）——每个关节朝哪个方向转才能撞到开关，是写在 `g_joints_init` 里的。

> 作者在 README 里承认了一个已知问题：**"上电时系统无法自动判断复位运动方向，导致复位过程存在不确定性"**——也就是复位方向是预设死的，如果机械臂断电时的姿态导致关节朝错误方向撞不到开关，复位可能卡住。这是第一代产品的局限，联调时如果遇到，手动把关节掰到开关附近再复位即可。

## 4. 软复位：不撞开关直接回位

`soft_reset` → `robot_joint_soft_reset()`：不依赖限位开关，**按当前角度算一条最短路径回到初始化角度**。

```c
for (int i = ROBOT_MAX_JOINT_NUM - 1; i >= 0; i--) {   // 关节6 → 关节1，全都要
    robot_update_current_angle(i);   // 从编码器读当前角度
    ret = robot_angle_map(current, min, max, &angle);   // 归一化到 0~360
    // 判断方向：目标角度 > 当前角度 → 逆时针；反之顺时针
    if (angle > g_joints_init[i].current_angle) dir = DIR_NEGATIVE;
    // 对 0~360° 关节：如果路程超过 180°，反向走更近
    if (min == 0 && max == 360 && fabs(angle - init) > 180) dir = -dir;
    robot_joint_rotate_to(i, dir, init_angle, ...);     // 转过去
}
```

关键点：
- 软复位**需要知道当前角度**——前提是编码器位置还有效（比如刚做完硬复位/没断过电）
- 对 0~360° 的全回转关节（关节 1、6），会判断**走哪边更近**（超过 180° 就反向）
- 初始角度就是 `g_joints_init` 里的 `current_angle`（关节 1：90°、关节 2：90°、关节 3：-90°、关节 4：0°、关节 5：90°、关节 6：0°）

## 5. zero 命令：自定义零点

`zero` 命令把**当前位置**设为零点：

```c
static int robot_zero_handle(float *param) {
    for (int i = 0; i < ROBOT_MAX_JOINT_NUM; i++) {
        Emm_V5_Reset_CurPos_To_Zero(i + 1);   // 6 个驱动器的当前位置全部清零
        vTaskDelay(10);
    }
    return pdPASS;
}
```

清零后，`soft_reset` 就是回到"你定义的这个零点的初始化姿态"。**先 `hard_reset`（建立物理零点）→ 再 `zero`（自定义逻辑零点）** 是常见组合。

## 6. 上电流程 Checklist

装好整机后，第一次上电建议按这个顺序（安全第一）：

- [ ] 确认供电：先只给主控板供电，**不接电机电源**
- [ ] 确认机械臂周围无杂物，各关节活动范围无障碍
- [ ] 打开串口工具（115200），看到启动日志
- [ ] 发 `hard_reset`，观察：关节 5→4→3→2→1 依次转动，每个都撞到限位开关后停下
- [ ] 发 `rel_rotate 1 30` 验证关节 1 能动
- [ ] 一切正常再接电机电源，进行正式联调（第 8 篇）

## 7. 常见问题

**Q: hard_reset 后某个关节还在一直转？**
可能该关节的限位开关没触发——检查开关接线（PD0~PD5 对应关节 1~5）、开关是否被结构件挡住、复位方向 `reset_dir` 是否设反。

**Q: 复位时关节撞开关的声音很大，正常吗？**
微动开关被撞到会"咔哒"一声，正常。但注意别让它长时间顶着开关转——代码里撞到就停，如果不停说明中断没触发（检查 EXTI 配置）。

**Q: 为什么软复位有时回不到初始位置？**
软复位依赖当前角度值。如果断电后编码器位置丢了、或之前没做过硬复位，当前角度可能是错的——先 `hard_reset` 再 `soft_reset`。

**Q: 关节 6 没有限位开关，它怎么找零点？**
它只能靠软复位（回初始化角度）或 `zero` 手动定义零点。它的 `hard_reset` 不在代码里（见上面的 `- 2` 细节），这是项目已知的简化。

## 8. 下一篇预告

[08 串口命令联调：rel_rotate / zero / 常见坑](/2026/08/19/ZERO机械臂学习-08-串口命令联调/) —— 复位完成后，逐个命令把机械臂"玩"起来：单关节转动、看日志、遇到的各种坑。
