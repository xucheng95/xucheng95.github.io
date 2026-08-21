---
title: ZERO 机械臂学习之路 11：远程控制（手柄遥控 / MQTT / ESP8266）
date: 2026-08-19 22:30:00
tags:
- 机械臂
- 手柄
- MQTT
- ESP8266
categories: ZERO机械臂
mathjax: false
---

> 这是「ZERO 机械臂学习之路」系列的第 11 篇。进阶阶段第一篇：让机械臂**脱离电脑的串口线**——先用游戏手柄遥控（作者已实现），再摸清 MQTT 远程控制的实现与现状（半成品，正好是开放的练手项目）。**里程碑：远程控制打通**。

## 1. 两种远程控制路径

| 方式 | 实现状态 | 原理 | 本篇 |
|------|---------|------|:---:|
| 游戏手柄遥控 | ✅ 已实现 | 电脑跑 Python 脚本，读手柄 → 串口发命令 | 第 2 节 |
| MQTT 网络远程 | ⚠️ 半成品 | ESP8266 WiFi 模块接 MQTT 服务器，手机/网页发指令 | 第 3、4 节 |

## 2. 手柄遥控：robot_joystick.py

仓库根目录有个 `robot_joystick.py`——**上位机脚本**（跑在电脑上，不是单片机里）。

### 工作流程

```
游戏手柄(USB) → pygame 读轴 → serial 串口发命令 → STM32 解析 → 机械臂运动
```

### 关键代码

```python
# robot_joystick.py
import pygame
import serial

# 连接手柄（第一个手柄）
joystick = pygame.joystick.Joystick(0)
# 打开串口
ser = serial.Serial(port, baudrate=115200, timeout=1)

# 进入遥控模式（重要！）
ser.write(b"remote_enable\n")

# 主循环：每 100ms 读一次手柄轴，拼成命令发出去
while True:
    axes = get_joystick_axes()   # 手柄所有轴的值，保留两位小数
    axes_str = "remote_event " + " ".join(map(str, axes)) + "\n"
    ser.write(axes_str.encode())   # 例如：remote_event 0.0 -0.0 0.0 -0.0 -1.0 -1.0
    time.sleep(0.1)                # 10Hz 频率
```

运行方式：

```bash
python robot_joystick.py -p COM3 -b 115200
```

退出时按 **Q**，脚本会发送 `remote_disable\n` 退出遥控模式。

### 主控侧解析（robot_cmd.c）

```c
static int robot_remote_event_handle(float *param) {
    // 遥控模式没开启就忽略
    if (!ROBOT_STATUS_IS(g_robot.status, ROBOT_STATUS_RMODE_ENABLE)) return pdPASS;

    // 6 个轴值 → 5 维速度指令（vx vy vz rx ry）
    float vx = -param[0] * ROBOT_REMOTE_MAX_VELOCITY;
    float vy =  param[1] * ROBOT_REMOTE_MAX_VELOCITY;
    float vz = (param[4] - param[5]) / 2 * ROBOT_REMOTE_MAX_VELOCITY;
    float rx = -param[3] * ROBOT_REMOTE_MAX_RPM;
    float ry =  param[2] * ROBOT_REMOTE_MAX_RPM;
    // 存入全局遥控结构体，控制任务周期读取
    g_remote_control.vx = vx; ... 
}
```

相关常量（`robot.h`）：

```c
#define ROBOT_REMOTE_MAX_VELOCITY  (20.0f)  /* 最大平移速度 20 mm/s */
#define ROBOT_REMOTE_MAX_RPM       (5.0f)   /* 最大旋转速度 5 rpm */
#define ROBOT_REMOTE_TIME_RESOLUTION (50)   /* 时间插值分辨率 50ms */
```

**本质**：手柄给的不是"目标位置"而是"速度"——遥感推多少，末端就以多快的速度移动，松手就停。这比位置控制简单直接，适合人肉操作。

> 代码注释里 `robot_remote_service` 任务被整段注释掉了——说明遥控从"独立任务"改成了"控制任务里周期读取 `g_remote_control`"。你可以读 `robot.c` 里的 `robot_pid_remote()` 看它怎么把速度积分成位置。

## 3. MQTT：让机械臂"上云"

### MQTT 是什么

**MQTT** 是物联网最常用的轻量级消息协议，基于**发布/订阅**模型：

```
                ┌─────────────┐
发布者 ──topic──→│ MQTT 服务器  │──topic──→ 订阅者
                └─────────────┘
```

- 有个中心**服务器（Broker）**，大家连它
- 消息带 **主题（topic）**，比如 `arm/change`
- 谁订阅了这个 topic，谁就能收到发到该 topic 的消息——**发布者和订阅者互不认识**，解耦

### ZERO 的 MQTT 链路

```
手机/网页/脚本 ──MQTT──→ 服务器 ──MQTT──→ ESP8266 WiFi模块 ──串口(USART3)──→ STM32
```

ESP8266 用 **AT 指令**完成一切（`esp8266_mqtt.c`）：

```c
AT+MQTTUSERCFG=0,1,"client_id","user","pass",0,0,""   // 配置账号
AT+MQTTCONN=0,"服务器地址",端口,0                       // 连接服务器
AT+MQTTSUB=0,"arm/change",0                            // 订阅主题
AT+MQTTPUB=0,"arm/change","消息内容",0,0                // 发布消息
```

主控收到订阅消息后解析（`robot_cmd.c` 的 `robot_mqtt_handle`）：

```c
sscanf(cmd, "+MQTTSUBRECV:0,\"arm/change\",%d,[MCU][%d][%f %f %f %f %f %f]", ...);
// 格式：[MCU][类型][6个参数] → 按类型分发到绝对转动/移动/同步
```

和串口命令一样，MQTT 命令也走 cmd_queue → robot_cmd_service → event_queue 的同一套流水线——**换的是"入口"，不变的是"处理"**。

## 4. MQTT 的现状：半成品，正好是练手项目

认真读代码你会发现 MQTT 部分**没有完全打通**，这反而是好事——它是绝佳的开放作业：

| 问题 | 代码位置 | 说明 |
|------|---------|------|
| 默认关闭 | `robot.h` | `#define ROBOT_MQTT_ENABLE 0U`——整个 MQTT 逻辑被编译开关关掉了 |
| 订阅主题是占位符 | `esp8266_mqtt.h` | `#define MQTT_TOPIC "xxxx"`——要自己填真实主题 |
| 订阅/解析主题不一致 | `esp8266_mqtt.c` / `robot_cmd.c` | 订阅用 `MQTT_TOPIC`，解析却写死 `"arm/change"` |
| 服务器地址未配置 | `esp8266_mqtt.c` | `esp8266_connect_mqtt(server, port, ...)` 的参数要自己填 |

**想打通它，你需要**：

1. 有一个 MQTT 服务器（本地 Mosquitto / 免费公共 Broker 如 EMQX 的公共实例 / 云服务器自建）
2. 把 `ROBOT_MQTT_ENABLE` 改为 1，填好 `MQTT_TOPIC`、服务器地址、账号密码
3. 写个发布端（Python `paho-mqtt` 库几行代码就能发 `[MCU][1][30 0 0 0 0 0]`）
4. 编译烧录，验证手机/电脑发的消息能让机械臂动

> 完成这个任务，你就同时练了：WiFi 模块驱动、AT 指令协议、MQTT 协议、串口数据流、以及嵌入式里的编译开关/条件编译——性价比极高。作者在 README 里列的计划功能（WEB 可视化平台、语音识别/合成、大模型接入）也都建立在打通 MQTT 之上。

## 5. 常见问题

**Q: 手柄连接不上 / 读不到轴？**
先跑 `pygame` 的手柄检测：`pygame.joystick.get_count()` 是否为 0。手柄要**先插好再运行脚本**（脚本只在启动时枚举一次手柄）。

**Q: 手柄能动但方向乱？**
轴映射问题。`robot_joystick.py` 把轴按顺序拼进 `remote_event`，主控按固定顺序解析（vx/vy/vz/rx/ry）。不同手柄轴顺序不同，要么改脚本里轴的顺序，要么改主控解析——两个文件对照着调。

**Q: 发 `remote_event` 没反应？**
先确认发了 `remote_enable`（`ROBOT_STATUS_RMODE_ENABLE` 没置位时，`remote_event` 直接被忽略，见 `robot_remote_event_handle` 第一行）。

**Q: MQTT 开了编译开关还是连不上？**
先单独测试 ESP8266：串口工具直接发 AT 指令（`AT` 应答 `OK`、`AT+CWMODE=1` 配网），把 WiFi 和 MQTT 链路逐段打通，再合入主程序——**分段调试**永远比整体调试快。

## 6. 本篇小结

远程控制的本质是**给机械臂增加新的命令入口**：

```
串口（已通）→ 手柄（已通）→ MQTT/网络（待你打通）
                  ↓
        同一个命令解析 + 事件驱动架构
```

架构的好处在这里体现：加一种输入方式，不动控制核心。打通 MQTT 后，机械臂就能被网页、手机、甚至大模型"远程指挥"了——这正是下一篇强化学习与大模型的入口。

## 7. 下一篇预告

[12 强化学习与大模型：TD3 / MuJoCo / 展望](/2026/08/19/ZERO机械臂学习-12-强化学习与大模型/) —— 系列终篇：让机械臂"自己学会"控制——MuJoCo 仿真、TD3 算法、奖励设计，以及大模型接入的展望。
