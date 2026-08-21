---
title: ZERO 机械臂学习之路 09：运动学联调（auto / 坐标系 / 路径插值）
date: 2026-08-19 22:10:00
tags:
- 机械臂
- 运动学
- 逆解
- 联调
categories: ZERO机械臂
mathjax: false
---

> 这是「ZERO 机械臂学习之路」系列的第 09 篇。第 02 篇学了理论（正逆运动学），这一篇看它们在真机上是怎样协作的：发一条 `auto x y z`，机械臂从"会转"到"末端自动到位"。**里程碑：末端按指令到达指定位置**。

## 1. auto 命令：一句话让末端动起来

```
auto 0 0 200
```

意思是：让末端移动到 `(x=0, y=0, z=200)` 的位置（单位 mm）。注意 README 里的提醒：**注意坐标方向**。

代码路径（`robot_cmd.c` → `robot_control_task`）：

```
"auto x y z" → robot_auto_handle() → robot_send_auto_event()
    ↓
event_queue → robot_control_task → ROBOT_AUTO_EVENT 分支
    ↓
robot_auto_move_interpolation()   ← ★ 本篇主角
```

## 2. auto 背后的四步流水线

`robot_auto_move_interpolation()`（`robot.c`）完整地串起了第 02 篇学的所有概念：

```c
static void robot_auto_move_interpolation(struct robot_event *event) {
    // ① 路径插值：把目标点拆成一条直线路径上的很多个中间点
    struct position *path = robot_path_interpolation_linear(target_pos, &path_size);

    // ② 逐点逆解：对每个路径点求 6 个关节角
    for (int i = 0; i < path_size; i++) {
        robot_kinematics_cal_T(T_0_6_reset, g_robot.T, &path[i]);   // 目标位置 → T 矩阵
        ret = robot_kinematics_inverse((float *)g_robot.T, &result[i * 6], false);
        robot_kinematics_joint_angle_update(&result[i * 6]);        // 供下一个点选最优解
    }

    // ③ PID 沿路径执行
    ret = robot_pid_run(path, path_size, result);

    // ④ 更新末端当前位置
    g_robot.cur_pos.x = path[path_size-1].x; ...
}
```

| 步骤 | 函数 | 干什么 | 对应理论 |
|------|------|--------|---------|
| ① | `robot_path_interpolation_linear` | 直线插值成路径点 | 轨迹规划 |
| ② | `robot_kinematics_cal_T` + `robot_kinematics_inverse` | 每点做逆解 | 逆运动学 |
| ③ | `robot_pid_run` | 沿路径走，PID 跟随 | 控制 |
| ④ | — | 更新当前位置 | — |

## 3. 坐标系：auto 的 x y z 是什么

这是最容易被绕晕的地方。看 `robot_kinematics_cal_T`：

```c
void robot_kinematics_cal_T(const float T_in[4][4], float T_out[4][4], struct position *pos) {
    memcpy(T_out, T_in, sizeof(float)*16);
    T_out[0][3] = pos->x + T_out[0][3];   // 只改平移列！
    T_out[1][3] = pos->y + T_out[1][3];
    T_out[2][3] = pos->z + T_out[2][3];
}
```

它在**复位姿态矩阵 `T_0_6_reset` 的基础上加上偏移**：

```c
const float T_0_6_reset[4][4] = {
    {0, -1, 0, 0},
    {0, 0, -1, -47.63},   // 复位时末端在 y=-47.63
    {1, 0, 0, 15.5},      // z=15.5
    {0, 0, 0, 1},
};
```

所以 `auto x y z` 的坐标是**相对复位姿态的偏移量**，不是绝对的世界坐标：

```
末端位置 = 复位位置(0, -47.63, 15.5) + (x, y, z)
```

> 代码注释里作者也写了："根据末端坐标的相对运动，获得末端的 T 矩阵（当前仅支持 x,y,z 的相对运动，后续支持旋转）"——也就是说目前 `auto` **只支持平移，不支持姿态旋转**，这是第一版的功能边界。

### DH 参数：T 矩阵背后的数字（兑现第 02 篇的承诺）

第 02 篇说过 DH 参数表"后面会细讲"，现在兑现。`robot.c` 里定义了完整的 DH 参数表，每行是 `[a, alpha, d, theta]` 四个参数：

```c
/* robot.c：机械臂各关节 DH 参数 */
const float D_H[6][4] = {
    {0,      0,        0,      M_PI/2},
    {0,      M_PI/2,   0,      M_PI/2},
    {200,    M_PI,     0,      -M_PI/2},   /* a2 = D_H[2][0] = 200   连杆2长度 */
    {47.63,  -M_PI/2,  -184.5, 0},         /* a3 = D_H[3][0] = 47.63 连杆3长度 */
    {0,      M_PI/2,   0,      M_PI/2},    /* d4 = D_H[3][2] = -184.5 连杆4偏距 */
    {0,      M_PI/2,   0,      0}
};
```

运动学求解用到的三个关键量都来自这张表（第 02 篇的 `robot_kinematics.c` 宏）：

```c
#define a2 D_H[2][0]   // 连杆2长度 = 200mm
#define a3 D_H[3][0]   // 连杆3长度 = 47.63mm
#define d4 D_H[3][2]   // 连杆4偏距 = -184.5mm
```

> **这些数字怎么来的？** 来自 SolidWorks 三维模型的实测尺寸（连杆长度/偏距是机械结构的几何属性），而逆解的公式则是由 MATLAB **符号推导**出来的：仓库 `3. Simulink/` 下的 `robot_kinematics.m` / `robot_kinematics_sym_v3_0.m` 就是推导过程（代码注释里也写了"各关节运动学逆解公式可参考 robot_kinematics_sym_v3_0.m"）。想搞懂第 02 篇那两个开方公式从哪来，读这份推导是最直接的路。

## 4. 路径插值：为什么不是一步到位

机械臂如果直接"瞬移"到目标点，电机要瞬间提速，会冲击、失步。所以先把目标点拆成小段：

```c
static struct position *robot_path_interpolation_linear(struct position *target, int *size) {
    // 从当前末端位置到目标位置做直线插值
    // 插值分辨率：1mm（ROBOT_INTERPOLATION_RESOLUTION）
    // 时间分辨率：100ms（ROBOT_INTERPOLATION_TIME_RESOLUTION）
}
```

两个关键常量（`robot.h`）：

```c
#define ROBOT_INTERPOLATION_TIME_RESOLUTION (100)   // 每个路径点间隔 100ms
#define ROBOT_INTERPOLATION_RESOLUTION      (1.0f)  // 路径点间距 1mm
```

效果：从起点到终点，每隔约 1mm 取一个点，每个点 100ms——机械臂"平滑地走过去"，而不是"跳过去"。

## 5. 逐点逆解 + 最优解选择

每个路径点都要做一次完整逆解。回顾第 02 篇：代码算出 **4 组候选解**，然后用两步筛出唯一解：

**第一步：关节限位过滤**（`robot_kinematics_joint_angle_map`）
把 0~360° 的角度翻折映射进每个关节的 `min_angle`~`max_angle`，翻不进去的解标记为无效（`result_invalid_mask`）。

**第二步：加权选最优**（`robot_kinematics_get_optimal_result`）

```c
diff = 0;
for (int j = 0; j < 6; j++) {
    diff += fabs(result[i][j] - g_current_joint_angle[j]) * joint_weight[j];
}
// 选 diff 最小的解
```

`joint_weight = {5, 3, 3, 1, 1, 1}`——**越靠近底座的关节权重越高**，因为大关节动一下代价大，优先保持它们不动。

> 还有个细节：每个点算完后 `robot_kinematics_joint_angle_update()` 把该点的解写回"当前角度"，下一个点选最优解时以它为准——保证**相邻路径点的解是连续的**，不会出现两个相邻点选了不同的 IK 分支导致关节大幅跳变。

## 6. PID 沿路径执行

逆解算出的是一串"关节角度目标"，`robot_pid_run` 把它们平滑地喂给电机。核心是 20ms 周期的位置 PID（`robot_pid_one_period`）：

```c
#define ROBOT_PID_KP  (10.0f)    // 比例
#define ROBOT_PID_KI  (0.002f)   // 积分
#define ROBOT_PID_KD  (0.0f)     // 微分：没用
#define ROBOT_PID_PERIOD (20)    // 20ms 一个控制周期
```

每个周期：
1. `robot_update_current_angle(j)`：从编码器读当前角度
2. 算误差 `error = target - current`
3. `v = Kp*error + Ki*∫error + Kd*Δerror`
4. 把速度指令通过 CAN 发给驱动器

路径跑完后打印每个关节的累计误差：

```
[jpint 1] ave_error:0.35
...
robot pid run finished!!
```

误差越小说明跟随越准。**如果误差大或末端到位慢，先调 `ROBOT_PID_KP`（往大调一点），再调 `ROBOT_PID_KI`**。

## 7. 联调实操

```
# 1. 复位
hard_reset
# 2. 先小幅度试：末端往 z 方向抬 200mm
auto 0 0 200
# 3. 看日志：每个路径点的坐标和逆解结果
#    [12] <0.00 0.00 178.44> result: 91.32 122.41 -91.05 0.00 90.00 0.00
# 4. 尝试 x/y 方向
auto 100 0 200
auto 0 100 200
```

**日志解读**：`[i] <x y z> result: θ1 θ2 θ3 θ4 θ5 θ6`——第 i 个路径点的末端坐标和对应的 6 个关节角。你可以把日志里的关节角代回第 02 篇的 Python 正运动学代码验证末端位置——**逆解算出来的角度，正解算回去应该就是路径点**，这是理解正逆运动学关系最好的实验。

## 8. 常见问题

**Q: 发 auto 后日志打印 `robot kinematics inverse failed`？**
目标点在工作空间之外（够不到）或姿态不可达。把 x/y/z 调小或换个方向试。这就是第 02 篇说的工作空间约束在起作用。

**Q: auto 移动的轨迹不是直线？**
理论上插值保证直线，但关节运动是"每个点分别逆解 + PID 跟随"，如果 PID 没跟上或某点逆解失败，轨迹会偏。误差大就先调 PID。

**Q: 末端到位后有抖动？**
PID 参数 Kp 太大或积分饱和。Kp 从 10 往下调试试，或检查机械旷量（README 里作者承认行星减速器旷量较大，存在晃动）。

**Q: 怎么确认坐标方向对不对？**
README 特别提醒"注意坐标方向"。可以发一个很小的位移（如 `auto 0 0 10`）观察末端往哪动，确认 x/y/z 的正方向与你的预期一致，再放大位移。

## 9. 本篇小结

`auto` 命令是整台机械臂的"集大成者"：

```
目标点(x,y,z) ─→ 直线插值(1mm/100ms) ─→ 逐点逆解(4解→限位→加权选优) ─→ PID(20ms) ─→ CAN ─→ 电机
```

到这里，机械臂的"大脑-神经-肌肉"全链路已经打通。下一篇回到"身体"本身：把整机装配的细节和坑讲清楚（按路线图，装配篇在这里补齐）。

## 10. 下一篇预告

[10 整机装配：3D 打印件 / 装配顺序 / 走线](/2026/08/19/ZERO机械臂学习-10-整机装配/) —— 从一箱零件到一台能动的机械臂：装配顺序、热熔螺母、同步带张紧、走线布线，以及装配中的常见坑。
