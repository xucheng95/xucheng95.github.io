---
title: ZERO 机械臂学习之路 05：主控与通讯（STM32 / CAN 总线 / 串口）
date: 2026-08-19 21:30:00
tags:
- 机械臂
- STM32
- CAN总线
- 串口
categories: ZERO机械臂
mathjax: false
---

> 这是「ZERO 机械臂学习之路」系列的第 05 篇。上一篇讲完了"肌肉"（电机与传动），这一篇讲"大脑"和"神经"：主控 STM32 怎么调度任务、串口怎么收你的命令、CAN 总线怎么跟 6 个电机对话。读完你会对整个 `robot` 工程的启动流程和消息流转有完整认识。

## 1. 主控：STM32F407VET6

整台机械臂的"大脑"是一块 **STM32F407VET6** 开发板（BOM 第 3 项）。为什么是它：

- **Cortex-M4F 内核**，168MHz，带硬件浮点单元——跑运动学逆解的那堆三角函数和矩阵运算没问题
- **自带 CAN 控制器**（关键外设！）——连接张大头驱动器全靠它
- 资源足够跑 **FreeRTOS + 运动学计算 + 串口/网络协议**，还有富余

从 `robot.ioc`（CubeMX 工程文件）可以看到用到的外设：

| 外设 | 引脚 | 用途 |
|------|------|------|
| CAN1 | PB8 / PB9 | 连接 6 个张大头驱动器（CAN 总线） |
| USART1 | — | 串口命令输入（电脑 ↔ 主控，115200） |
| USART3 | — | 接 ESP8266 WiFi 模块（MQTT 远程，默认关闭） |
| GPIO（EXTI 中断） | PD0 ~ PD5 | 6 个关节的限位开关输入 |
| RNG | — | 硬件随机数（给 FreeRTOS 用） |
| DMA | USART3 RX | 串口 DMA 传输 |

## 2. 上电后发生了什么：启动流程

代码从 `Core/Src/main.c` 开始，一步步走到你的机械臂"活"起来：

```
main()
 ├─ HAL_Init()               ← HAL 库初始化
 ├─ SystemClock_Config()     ← 配置时钟（168MHz）
 ├─ MX_GPIO_Init()           ← GPIO：限位开关引脚、LED 等
 ├─ MX_DMA_Init()            ← DMA
 ├─ MX_CAN1_Init()           ← CAN 控制器（波特率、过滤器）
 ├─ MX_RNG_Init()            ← 随机数
 ├─ MX_USART1_UART_Init()    ← 串口1（命令口）
 ├─ MX_USART3_UART_Init()    ← 串口3（WiFi 口）
 ├─ MX_FREERTOS_Init()       ← FreeRTOS 内核初始化
 ├─ osKernelStart()          ← 启动调度器！
 └─ StartDefaultTask（第一个任务）
     └─ robot_init()         ← 创建机器人的核心任务和队列
```

`robot_init()`（`robot.c`）做的事很清晰：

1. 把 `g_joints_init`（关节配置表）和 `T_0_6_reset`（复位姿态矩阵）拷贝进全局结构体
2. 创建 **事件队列** `event_queue`（容量 20，`ROBOT_MAX_EVENT_NUM`）
3. 创建 **命令队列** `cmd_queue`（容量 50，`ROBOT_CMD_MAX_NUM`）
4. 创建两个任务：`robot_control_task`（实时优先级 3）和 `robot_cmd_service`（优先级 2）

## 3. FreeRTOS 任务架构：两个队列 + 两个任务

整个软件就是**事件驱动**：任何输入（串口命令、限位开关、手柄、MQTT）都变成一条消息，扔进队列，由对应任务处理：

```
              ┌──────────────────────────────┐
 串口/USART1 ─┤                              │
 手柄/MQTT ───┤→ cmd_queue（命令字符串）      │
              │        ↓                     │
              │  robot_cmd_service 任务       │ ← 解析字符串 → 查命令表
              │        ↓                     │
              │  event_queue（结构化事件）     │
              │        ↓                     │
              │  robot_control_task 任务      │ ← 真正的控制：运动学/PID/CAN
              └──────────────────────────────┘
```

| 对象 | 代码 | 作用 |
|------|------|------|
| 命令队列 | `robot_cmd.c` → `robot_cmd_service` | 存放原始命令字符串，串口中断里塞进来 |
| 命令解析 | `robot_uart1_handle()` | `sscanf` 拆出命令名 + 参数，查 `robot_uart1_cmd_table` |
| 事件队列 | `robot.c` → `robot_control_task` | 存放结构化事件（转动/复位/移动） |
| 控制任务 | `robot_control_task()` | `xQueueReceive` 死循环，按事件类型分发 |

为什么拆两层？**解析字符串慢、控制要实时**。把慢的解析放进低优先级任务，高优先级控制任务只处理结构化事件，互不拖累——这是嵌入式里的标准做法。

## 4. 串口：你发命令的第一条通道

### 物理层

电脑的 USB 口接一块 **USB 转 TTL 模块**（BOM 之外，约 10 元），TTL 端三根线连到开发板的 USART1（TX、RX、GND）。**注意 TX 接 RX、RX 接 TX**，交叉接，这是新手第一大坑。

### 代码层

`usart.c` 里串口是**逐字节中断接收**：

```c
// usart.c
HAL_UART_Receive_IT(&huart1, (uint8_t *)uart1_rx_buff, 1);  // 每收 1 个字节触发一次

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) {
    if (huart->Instance == USART1) {
        if (uart1_rx_buff[uart1_rx_pos] == '\n') {   // 收到换行符 = 一条命令结束
            uart1_rx_buff[uart1_rx_pos] = '\0';
            robot_cmd_send_from_isr(uart1_rx_buff, CMD_TYPE_UART1);  // 塞进命令队列
            uart1_rx_pos = 0;
        } else {
            uart1_rx_pos += 1;   // 没结束就继续攒
        }
        HAL_UART_Receive_IT(&huart1, (uint8_t *)&uart1_rx_buff[uart1_rx_pos], 1);
    }
}
```

**要点**：
- 命令以 `\n`（换行符）结束——串口工具里发命令时记得勾上"发送新行"
- 命令最长 128 字节（`ROBOT_CMD_LENGTH`），一条命令一行
- 中断里只做"塞队列"这一件轻量事，解析交给 `robot_cmd_service`——**中断里绝不能干重活**

### 命令解析（robot_cmd.c）

```c
ret = sscanf(cmd, "%19s %f %f %f %f %f %f", event_type, &param[0], ..., &param[5]);

for (int i = 0; robot_uart1_cmd_table[i].event_type != NULL; i++) {
    if (strcmp(event_type, robot_uart1_cmd_table[i].event_type) == 0) {
        ret = robot_uart1_cmd_table[i].cmd_func(param);   // 调用对应处理函数
    }
}
```

命令表（`robot_uart1_cmd_table[]`）：

| 命令 | 参数 | 功能 |
|------|------|------|
| `rel_rotate` | 关节号 角度 | 指定关节相对转动（支持负数） |
| `auto` | x y z | 末端移动到指定位置 |
| `hard_reset` | — | 硬复位：撞限位开关找零点 |
| `soft_reset` | — | 软复位：回初始化角度 |
| `zero` | — | 把当前位置设为零点 |
| `remote_enable` / `remote_disable` / `remote_event` | — | 手柄遥控（第 11 篇细讲） |

## 5. CAN 总线：主控和电机的"高速公路"

### CAN 是什么

**CAN（Controller Area Network）** 是工业界最常用的现场总线之一，两根线（CANH / CANL）就能挂几十个设备，大家**共用一条总线通信**。特点：

- **差分信号**：两根线上电压差传数据，抗干扰强——车间里电机狂转也不会丢数据
- **多主架构**：任何节点都能主动发消息，靠**仲裁**解决冲突（ID 小的优先）
- **报文式**：不是点对点，而是广播——带上 ID，谁关心谁收

对机械臂来说：**6 个驱动器 + 主控 = 7 个节点挂一条双绞线**，主控给每个驱动器发"转到某角度"的指令，驱动器回"我现在的位置/状态"。

```
STM32F407 ──CANH/CANL（双绞线）── 驱动器1 ── 驱动器2 ── ... ── 驱动器6
   │  PB8/PB9                                                      │
   └────────────── 120Ω 终端电阻 ───────────── 120Ω 终端电阻 ────────┘
```

> **终端电阻**：总线两端各接一个 120Ω 电阻（代码里驱动拓展板上带）。没有终端电阻，高速通信会因信号反射出错——联调时"时好时坏"先查它。

### STM32 侧配置（can.c）

```c
hcan1.Init.Prescaler = 14;        // 预分频 → 决定波特率
// 过滤器：全部放行，只用一个 FIFO0
canFilter.FilterFIFOAssignment = CAN_RX_FIFO0;
```

收消息的回调：

```c
void HAL_CAN_RxFifo0MsgPendingCallback(CAN_HandleTypeDef *hcan) {
    HAL_CAN_GetRxMessage(hcan, CAN_RX_FIFO0, ...);
    // 校验：DLC==3 且数据是 0xfd 0x9f 0x6b → 张大头驱动器的响应帧
    if ((can.CAN_RxMsg.DLC == 3) && (can.rxData[0] == 0xfd) && ...) { ... }
}
```

### 张大头私有协议（Emm_V5.c）

张大头驱动器用的是**私有 CAN 协议**，每帧有固定的"功能码 + 辅助码 + 校验字节"结构。`Emm_V5.c` 封装了常用操作：

```c
Emm_V5_Reset_CurPos_To_Zero(addr);   // 功能码 0x0A：把当前编码器位置清零
Emm_V5_Reset_Clog_Pro(addr);         // 功能码 0x0E：解除堵转保护
Emm_V5_Read_Sys_Params(addr, ...);   // 读系统参数：位置/速度/电压/PID 等
```

> 这也是为什么 BOM 里驱动器**必须买张大头 CAN 协议版**——协议层（`Emm_V5.c`）是照着张大头的私有帧格式写的，换品牌就得重写这层。这是复刻项目最容易买错的地方，再次提醒。

## 6. 一条命令的完整旅程

以 `rel_rotate 1 30`（关节 1 转 30°）为例，串起来看：

```
你输入 "rel_rotate 1 30\n"
   │
   ▼
USART1 中断收满一行 → robot_cmd_send_from_isr() → cmd_queue
   │
   ▼
robot_cmd_service 任务 → sscanf 解析 → 查命令表 → robot_rel_rotate_handle()
   │  把"关节1 转30°"封装成事件，塞进 event_queue
   ▼
robot_control_task 任务 → ROBOT_JOINT_REL_ROTATE 分支 → robot_joint_rotate_to()
   │  按减速比换算电机角度 → 走 PID → 算 CAN 指令
   ▼
can.c → HAL_CAN_AddTxMessage() → CAN 总线
   ▼
张大头驱动器 → 电机转动 30°
```

## 7. 动手实验（需要硬件）

1. 用串口工具（XCOM / Sscom / Serial Studio 均可）打开 USB 转 TTL 对应的 COM 口，波特率 **115200**
2. 发 `hard_reset`，观察机械臂各关节依次转动、撞限位开关停下（这就是第 7 篇要细讲的复位）
3. 发 `rel_rotate 1 30`，关节 1 应转 30°；发 `rel_rotate 1 -30` 转回来
4. 如果没反应，串口里应该能看到日志（`robot control task runing!!!` 等）——日志是最强的调试工具

## 8. 常见问题

**Q: 串口发命令没反应？**
先看串口工具是否勾了"发送新行"（代码靠 `\n` 判定命令结束）；再看 TX/RX 是否交叉接；最后看波特率是否 115200。

**Q: CAN 总线为什么适合机械臂而不是 USB？**
USB 是主从结构（电脑是主机），CAN 是真正的多主总线，任意节点可主动通信；而且差分信号抗干扰、线少（2 根）、可靠性高——汽车、工业设备到处是它。

**Q: 为什么命令解析要单独开一个任务？**
字符串解析（`sscanf`、`strcmp`）慢且会阻塞，放高优先级任务会拖累实时控制。队列解耦：中断只入队、低优先级任务慢慢解析、高优先级任务只管控制。

## 9. 下一篇预告

[06 嵌入式工具链：CubeMX / CubeIDE / 编译烧录](/2026/08/19/ZERO机械臂学习-06-嵌入式工具链/) —— 把代码跑起来的第一步：装环境、开工程、编译、烧录，看到串口日志的那一刻就成功了一半。
