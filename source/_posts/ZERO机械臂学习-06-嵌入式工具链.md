---
title: ZERO 机械臂学习之路 06：嵌入式工具链（CubeMX / CubeIDE / 编译烧录）
date: 2026-08-19 21:40:00
tags:
- 机械臂
- STM32
- 嵌入式
- 工具链
categories: ZERO机械臂
mathjax: false
---

> 这是「ZERO 机械臂学习之路」系列的第 06 篇。上一篇认识了主控和通讯，这一篇把"编译烧录"这条工具链打通：装什么软件、工程目录是什么结构、怎么编译、怎么烧录、怎么用串口日志调试。**里程碑：看到串口里出现 `robot control task runing!!!`**。

## 1. 需要安装的软件清单

| 软件 | 作用 | 备注 |
|------|------|------|
| STM32CubeMX | 图形化配置外设/时钟/引脚，生成初始化代码 | 打开 `robot.ioc` 看工程配置 |
| STM32CubeIDE | 官方 IDE，编译 + 调试 + 下载一体 | 基于 Eclipse |
| ST-Link 驱动 | 让电脑识别 ST-Link 烧录器 | 买 ST-Link 时店家会给 |
| 串口工具 | XCOM / Sscom / Serial Studio | 看日志、发命令 |
| VSCode（可选） | 作者实际写代码的地方 | 只写不改配置，CubeIDE 当"工具人" |

> 作者原话：**"STM32CubeMX 用于硬件外设初始化，CubeIDE 我一般用来 DEBUG 和程序下载，敲代码我还是喜欢用 VSCode，那两兄弟就是工具人。"**——你也可以这么搭配。

## 2. 工程目录结构（先认路）

`2. Software/robot/` 是一个标准的 **STM32CubeIDE 工程**：

```
robot/
├── robot.ioc              ← ★ CubeMX 工程文件：所有外设/引脚/时钟配置
├── .cproject / .project   ← CubeIDE 工程元数据
├── STM32F407VETX_FLASH.ld ← 链接脚本（代码放在 Flash 的哪、RAM 用多少）
├── robot Debug.launch     ← 调试启动配置
├── Core/                  ← ★ 核心代码（你真正要读的地方）
│   ├── Inc/               ←   头文件（robot.h、robot_kinematics.h、Emm_V5.h…）
│   └── Src/               ←   源文件（robot.c、robot_kinematics.c、robot_cmd.c…）
├── Drivers/               ← STM32 HAL 库（ST 官方，一般不动）
└── Middlewares/           ← FreeRTOS 内核（一般不动）
```

**核心代码在 `Core/`，HAL 库和 FreeRTOS 在 `Drivers/` 和 `Middlewares/`**——你 90% 的时间都在 `Core/Src` 里。

## 3. 用 CubeMX 看工程配置

用 STM32CubeMX 打开 `robot.ioc`，你会看到作者做好的全部硬件配置（**只读，别乱改**）：

- **Pinout 视图**：PD0~PD5 上挂着 6 个 GPIO 输入（限位开关，带 EXTI 中断）、PB8/PB9 是 CAN1、USART1/USART3、RNG
- **Clock 视图**：外部晶振 → PLL → 168MHz 主频
- **Middleware 视图**：FreeRTOS（CMSIS-RTOS2 接口，作者用的是 `osThreadNew` 而不是裸的 `xTaskCreate`）

> 想动手改外设（比如换串口引脚）就在这改，改完 **Generate Code** 重新生成——但生成前最好备份，CubeMX 会把 `Core/Src` 里你写的手工代码和它生成的混在一起，重新生成可能覆盖（其实 `USER CODE BEGIN/END` 注释块内的代码不会动，规矩就是**手写代码必须写在这两个注释之间**）。

## 4. 编译

在 CubeIDE 里：右键工程 → **Build Project**（或点锤子图标）。底层实际调用的是 `arm-none-eabi-gcc` 交叉编译器，CubeIDE 自带。

编译产物：

| 文件 | 说明 |
|------|------|
| `robot.elf` | 带调试信息的完整镜像（调试用） |
| `robot.bin` / `.hex` | 纯固件镜像（烧录用） |

第一次编译会慢一些（HAL 库全量编译），之后是增量编译，几秒到十几秒。**编译报错 90% 是你改代码时括号/分号/类型写错**，看错误信息定位到 `Core/Src` 里的文件即可。

## 5. 烧录

### 硬件连接

ST-Link 烧录器通过 **SWD 四线**连接开发板：

```
ST-Link         开发板
  SWDIO  ────→  SWDIO
  SWCLK  ────→  SWCLK
  GND    ────→  GND
  3.3V   ────→  3.3V（可选，开发板一般自己供电）
```

### 软件操作

1. 开发板上电（注意：机械臂没装好的时候**别接电机电源**，先只给主控板供电）
2. CubeIDE 里：Run → **Debug**（或 Download），选择 `robot Debug.launch` 配置
3. 烧录完成后按 **F8（Resume）** 或直接断电重上电，程序开始跑

### 验证：串口日志

上电后，打开串口工具（115200），如果一切正常你会看到：

```
robot control task runing!!!
```

——这就是里程碑达成的声音。如果还发过 `rel_rotate` 之类命令，日志里会有对应的处理记录（`[joint_id: 1] ROBOT_JOINT_REL_ROTATE 30.000000`）。

## 6. 烧录常见坑

**Q: 烧录报 `No ST-Link detected`？**
驱动没装好，或 ST-Link 的 USB 线只供电不通数据。换一根数据线试试。

**Q: 烧录报 `Error: Target not connected` / `No target connected`？**
SWD 四根线没接对（SWDIO/SWCLK 别接反），或目标板没上电，或芯片被锁死（Debug 模式下 `NRST` 拉低试试，CubeIDE 里选 Connect under reset）。

**Q: 烧完没反应，串口也没日志？**
先确认串口工具连的是 USART1 的 TX/RX（不是别的口）、波特率 115200、线没接反；再看编译时有没有报错没烧进去。

**Q: 为啥我改的代码没生效？**
很可能改了没重新编译，或烧录的是旧的 `.elf`。养成习惯：**改完 → Build → Debug/Download → 看日志**。

## 7. 第一个属于自己的改动

把"编译烧录"链路跑通后，做一个小改动验证你掌握了全流程（很简单，改完能编译烧录、能从日志看到变化即可）：

1. 打开 `robot_cmd.c`，找到 `robot_zero_handle` 里的 `LOG("robot reset zero.\n")`
2. 改成 `LOG("robot reset zero. [my first edit]\n")`
3. Build → 烧录 → 发 `zero` 命令 → 串口看到新日志

> 这一步的意义不在改了什么，而在**建立了"改代码 → 编译 → 烧录 → 验证"的闭环**。后面所有联调都靠这个循环。

## 8. 本篇小结

| 环节 | 工具 | 关键点 |
|------|------|--------|
| 看配置 | STM32CubeMX | `robot.ioc`，只读别乱改 |
| 改代码 | VSCode / CubeIDE | 手写代码放 `USER CODE BEGIN/END` 之间 |
| 编译 | CubeIDE Build | 底层 arm-none-eabi-gcc |
| 烧录 | ST-Link + SWD | 四线：SWDIO/SWCLK/GND/3.3V |
| 验证 | 串口工具 115200 | 看到 `robot control task runing!!!` |

## 9. 下一篇预告

[07 上电与复位：限位开关 / 硬复位 / 软复位](/2026/08/19/ZERO机械臂学习-07-上电与复位/) —— 机械臂断电后"失忆"了怎么办？限位开关怎么帮它找回零点？硬复位和软复位有什么区别？
