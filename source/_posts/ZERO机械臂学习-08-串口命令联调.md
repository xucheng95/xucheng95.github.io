---
title: ZERO 机械臂学习之路 08：串口命令联调（rel_rotate / zero / 常见坑）
date: 2026-08-19 22:00:00
tags:
- 机械臂
- 串口
- 联调
categories: ZERO机械臂
mathjax: false
---

> 这是「ZERO 机械臂学习之路」系列的第 08 篇。复位学会了，这一篇进入正式联调：用**串口命令**把每个关节玩转——`rel_rotate` 单关节转动、`zero` 设零点，并系统整理联调中 90% 会遇到的坑。**里程碑：每个关节都能按指令精确转动**。

## 1. 联调前的准备

- [ ] 整机装配完成（没装完可以先只测主控 + 1 个电机）
- [ ] 第 7 篇的上电流程跑通，`hard_reset` 正常
- [ ] 串口工具（XCOM / Sscom），波特率 **115200**，勾选"发送新行"
- [ ] 6 个驱动器的 CAN 地址确认正确（张大头驱动器用拨码/软件设置地址 1~6，与 `Emm_V5.c` 里的 `addr` 对应）

## 2. 命令速查表（再复习一遍）

| 命令 | 参数 | 功能 | 联调用途 |
|------|------|------|---------|
| `hard_reset` | — | 硬复位 | 开机必做 |
| `soft_reset` | — | 软复位 | 回初始姿态 |
| `zero` | — | 当前位置设为零点 | 自定义零点 |
| `rel_rotate` | `关节号 角度` | 相对转动（支持负数） | ★ 单关节测试 |
| `auto` | `x y z` | 末端移动到指定位置 | 第 9 篇细讲 |

## 3. 逐条命令实操

### 3.1 先做一次硬复位

```
hard_reset
```

观察串口日志，每个关节复位时会打印（`robot_joint_reset` 里）：

```
joint 1 limit switch already happend       ← 开关本来就顶着，直接清零
```

或复位过程中中断触发：

```
joint limit switch happened, joint id: 2    ← EXTI 中断触发了
```

### 3.2 rel_rotate：单关节相对转动

```
rel_rotate 1 30      ← 关节 1 正转 30°
rel_rotate 1 -30     ← 关节 1 反转 30°
rel_rotate 2 10      ← 关节 2 转 10°
```

代码路径（`robot_cmd.c` → `robot.c`）：

```c
static int robot_rel_rotate_handle(float *param) {
    uint32_t joint_id = (uint32_t)param[0];
    return robot_send_rel_rotate_event(joint_id, param[1]);
}
```

事件被 `robot_control_task` 处理，最终走 `robot_joint_rotate_to`：

```c
robot_joint_rotate_to(joint_id, DIR_POSITIVE, angle,
    ROBOT_JOINT_DEFAULT_VELOCITY,   // 默认速度 10 rpm
    ROBOT_JOINT_DEFAULT_ACCELERATION, // 默认加速度 200
    false);   // 相对模式
```

日志里你会看到：

```
[joint_id: 1] ROBOT_JOINT_REL_ROTATE 30.000000
```

### 3.3 zero：自定义零点

```
zero
```

把所有驱动器的编码器位置清零（`Emm_V5_Reset_CurPos_To_Zero`）。清零后，再发 `rel_rotate` 就以新零点为基准。

> **建议的零点流程**：先 `hard_reset`（物理零点，撞限位）→ 再按需 `zero`（逻辑零点，比如你想让某个姿态作为"初始姿态"）。

## 4. 联调核心坑位（按出现频率排序）

### 坑 1：命令没反应，串口无日志

- 串口工具没勾"发送新行"（代码靠 `\n` 判定命令结束）——**最最常见**
- TX/RX 接反
- 波特率不是 115200

### 坑 2：转动方向反了

发 `rel_rotate 1 30`，关节却转了 -30°。修复（第 4 篇讲过）：

```c
// robot.c 关节配置表：正方向对应的电机旋转方向
{90, MOTOR_DIR_CCW, 50, ..., 0, 360, DIR_NEGATIVE},  /* 关节1 */
```

把 `MOTOR_DIR_CCW` 改成 `MOTOR_DIR_CW`（或反过来），重新编译烧录。

### 坑 3：角度不准（转 90° 实际转 89°）

**减速比没对齐**。`g_joints_init` 里的 `reduction_ratio`（50 / 50.89 / 51 / 26.85）必须和实际减速器一致：

```c
uint32_t steps = fabs(round(rel_angle * joint->reduction_ratio * 3200 / 360));
```

这个公式把"关节角度 × 减速比 × 每圈步数(3200)"换算成电机步数——减速比错一个，角度全错。校准方法：实测"电机转 N 圈、关节转多少度"，反推 `reduction_ratio` 填进去。

### 坑 4：某个关节不动 / 乱动

- 检查该关节驱动器的 **CAN 地址**是否与代码里的 `addr` 对应（`Emm_V5_*` 函数第一个参数）
- 检查 CAN 总线接线：双绞线是否串联、**终端电阻**是否在位
- 检查该关节的限位状态：如果 `ROBOT_STATUS_LIMIT_HAPPENED` 还挂着，关节可能拒绝转动（复位后未清状态位）

### 坑 5：电机堵转保护

负载过大或卡住时，闭环驱动器会进入堵转保护。代码里有专门的解除函数：

```c
Emm_V5_Reset_Clog_Pro(addr);   // 解除堵转保护
```

联调时关节被卡住后再发命令没反应，先解除堵转保护，再排查机械卡顿。

### 坑 6：日志乱码

波特率不对（常见 9600/57600/115200 混用），或串口工具编码不是 UTF-8/GBK。ZERO 日志是中文，选对编码。

## 5. 一个完整的联调脚本（照着发）

```bash
# 1. 复位
hard_reset
# 2. 确认每个关节能转（依次测 1~6）
rel_rotate 1 30
rel_rotate 1 -30
rel_rotate 2 30
rel_rotate 2 -30
...
rel_rotate 6 30
rel_rotate 6 -30
# 3. 设零点
zero
# 4. 软复位验证回到零点姿态
soft_reset
```

> 每个关节"转 30° 再转回 -30°"是最安全的单关节验证。**转之前先想清楚这个关节的活动范围**（比如关节 2 只能 90°~180°），别把机械臂掰到限位之外。

## 6. 常见问题

**Q: rel_rotate 的角度是相对还是绝对？**
**相对**——在当前位置基础上转指定角度，支持负数（反向）。绝对定位由 `auto`（末端位置）和软复位（关节绝对角度）承担。

**Q: 为什么有的关节转起来很慢？**
默认速度 `ROBOT_JOINT_DEFAULT_VELOCITY = 10 rpm`，对所有关节一样。负载大的关节（关节 1）转速慢是正常的；如果想单独调，可以改配置表或后续用高级命令。

**Q: 联调时电机抖动/异响？**
检查电机电流设置、驱动器细分设置，或机械装配卡涩（同步带太紧、螺丝过紧都会导致）。

## 7. 本篇小结

单关节联调的本质是验证三件事：

```
命令解析正确（串口 → cmd_queue → 命令表）
    ↓
电机方向正确（MOTOR_DIR_* 和实际接线一致）
    ↓
角度换算正确（reduction_ratio 和实际减速器一致）
```

三件事都对了，机械臂就会"指哪打哪"。下一步把多个关节协同起来——用 `auto` 命令让**末端**移动到指定空间位置，进入运动学联调。

## 8. 下一篇预告

[09 运动学联调：auto 命令 / 坐标系 / 路径插值](/2026/08/19/ZERO机械臂学习-09-运动学联调/) —— 从"单关节能转"到"末端自动到位"：`auto x y z` 背后是路径插值 + 逆解 + PID 的全链路。
