# ps2_hal_fixed · 麦轮小车（ps2_hal 的修正版，独立工程）（STM32F103C8 + DRV8833 ×2 + PS2 手柄）

四个麦克纳姆轮 + 三个舵机 + PS2 无线手柄。本工程是 `ps2_hal` 的**修正版**：
引脚与混控和已经跑通的实车完全一致，改的是"代码里说错的标签、漏掉的校验、
不安全的限速、没有看门狗"这几类问题。

> **为什么用 ps2_hal 而不是 ps2_fixed**
> 工程目录：`ps2_hal_fixed`（原来的 `ps2_hal` 一个字都没动）。
> 修好之后两者的**固件逻辑完全相同**（因为真实接线只有一套：PB6~PB9 是左板、
> PA0~PA3 是右板）。区别只在工程外壳：本工程沿用 ps2_hal 原来的
> `MDK-ARM/ps2.uvprojx` + `ps2.uvoptx`，而那份 `.uvoptx` 里**已经配好了 ST-Link
> 与 Flash 算法**，所以打开就能烧，不会遇到 "No ULINK2/ME Device found"。
>
> **唯一一处有意的行为变化**：原来的"限速 `MAX_SPEED=50`"是假的——斜推摇杆时
> `50+50=100`，两个轮子会跑到 100% 占空比。本版改成等比归一，50 真的守住 50，
> 所以**斜向/组合输入的最高速会比以前低一半**（纯前进/后退不变）。
> 想要旧手感就把 `main.c` 的 `MAX_SPEED` 改 `100`（那时归一是空操作，但也没有限速了）。

每处改动的原因、位置、验证方法见 [CHANGELOG.md](CHANGELOG.md)。

## 引脚总表（唯一依据）

| 功能 | 引脚 | 定时器 / 说明 |
|---|---|---|
| **左前轮 M1** | **PB8 / PB9** | TIM4_CH3/CH4，1kHz（左板 **B** 路） |
| **左后轮 M2** | **PB6 / PB7** | TIM4_CH1/CH2，1kHz（左板 **A** 路） |
| **右前轮 M3** | **PA0 / PA1** | TIM2_CH1/CH2，1kHz（右板 **A** 路） |
| **右后轮 M4** | **PA2 / PA3** | TIM2_CH3/CH4，1kHz（右板 **B** 路） |
| 左驱动板 STBY | PB11 | GPIO，高电平使能 |
| 右驱动板 STBY | PB10 | GPIO，高电平使能 |
| 舵机 MG995 | PA8 | TIM1_CH1，50Hz |
| 舵机 SG90-1 | PB0 | TIM3_CH3，50Hz |
| 舵机 SG90-2 | PB1 | TIM3_CH4，50Hz |
| PS2 接收器 DATA / CMD / CS / CLK | PA4 / PA5 / PA6 / PA7 | 软件 SPI |
| 串口调试 TX / RX | PA9 / PA10 | USART1，115200 8N1 |
| 下载 SWDIO / SWCLK | PA13 / PA14 | SWD（JTAG 已关，释放 PB3/PB4/PA15） |

⚠️ **左板的 A/B 两路是"前后交叉"接的**（A 路 → 左后轮，B 路 → 左前轮），
右板是"前后正常"（A 路 → 右前轮，B 路 → 右后轮）。`motor.h` 已按真实轮位映射好——
**这张表改一个字，车的行为就变，改前先把轮子悬空核对**。

## 驱动接线

**左板 DRV8833（PB6~PB9 + PB11）**：`AIN1→PB6 AIN2→PB7`（**左后**）、
`BIN1→PB8 BIN2→PB9`（**左前**）、`STBY→PB11`、`VM→电池+`
**右板 DRV8833（PA0~PA3 + PB10）**：`AIN1→PA0 AIN2→PA1`（右前）、
`BIN1→PA2 BIN2→PA3`（右后）、`STBY→PB10`、`VM→电池+`

## 四种模式（`Core/Src/main.c` 顶部改 `APP_MODE`）

| 值 | 用途 |
|---|---|
| **2（默认）** | 手柄遥控：麦轮行驶 + 舵机 |
| 1 | 电机自检：M1~M4 轮流正转 1 秒 / 反转 1 秒（串口打印真实轮位） |
| 3 | 舵机自检：三个舵机 10°↔170° 来回扫 |
| 4 | **麦轮动作自检**：前进/后退/右平移/左平移/顺时针/逆时针 依次各 2 秒，动作间停 0.3 秒，串口打印四轮目标值 |

也可用编译选项覆盖：`-DAPP_MODE=4`。

### 模式 2 操作

| 操作 | 动作 |
|---|---|
| 右摇杆 上下 | 前进 / 后退 |
| 右摇杆 左右 | 左右平移 |
| 左摇杆 左右 | 原地转 |
| 十字键 ↑ ↓ | MG995 −1° / +1°（按住连发） |
| 十字键 ← → | MG995 −5° / +5° |
| L1 / R1 | SG90-1 −5° / +5° |
| L2 / R2 | SG90-2 −5° / +5° |
| START | 三个舵机回中 90° |
| SELECT | 电机急停 / 恢复 |

### 混控公式

```c
FL = vx - vy - w      FR = vx + vy + w
RL = vx + vy - w      RR = vx - vy + w
```
`vx` 前为正、`vy` 右为正、`w` 顺时针为正，标准 X 型。
- 纯 `vy`：`FL` 与 `RR` 同向、`FR` 与 `RL` 同向 → **真横移**
- 纯 `w` ：左侧两轮反向、右侧两轮同向 → **原地转**

> 换轮子/改辊子朝向（X 型 ↔ O 型）时，正确改法是**把 `SIGN_VY` 和 `SIGN_W` 同时改 −1**，
> 不要改混控宏。验证用 `APP_MODE=4`：横移必须**对角同号**。

## 标定

- 轮子转向反了：`Core/Inc/motor.h` 的 `MOTOR_INVERT_*` 改 `1`（**每个宏后面标了真实轮子**）
- 摇杆方向反了：`main.c` 的 `SIGN_VX / SIGN_VY / SIGN_W`（vy 与 w 要同进同退）
- 速度上限：`main.c` 的 `MAX_SPEED`（默认 50，混控等比归一，保证不超）

## 电源

- 动力：2×18650（7.4V）直接进两块 DRV8833 的 VM
- 舵机：**独立 5V 降压模块**，不要从开发板取电；5V/GND 间并 470~1000µF 电容
- 单片机：ST-Link 3.3V 或独立 5V→3.3V
- **所有 GND 必须共地**

## 编译与烧录

- 打开 `MDK-ARM/ps2.uvprojx` 编译；产物 `MDK-ARM/ps2/ps2.hex`。
- 烧录：**直接按 F8（Download）即可**，本工程的 `ps2.uvoptx` 里已经配好
  ST-Link + `STM32F10x_128.FLM`。开机横幅是
  `===== ps2_hal (fixed) : 4 MECANUM + 3 SERVO + PS2 =====`，可用它确认烧进去的是新固件。

### 关于 `.uvoptx`（踩过的坑，务必看）

Keil 把**烧录配置存在同名的 `.uvoptx` 里**，不在 `.uvprojx`：

| 文件 | 负责什么 |
|---|---|
| `ps2.uvprojx` | 源文件、宏定义、优化等级、include、输出名 |
| `ps2.uvoptx` | **调试器（ST-Link）、Flash 算法、"Reset and Run"、断点** |

所以：
1. **不要删 `.uvoptx`，也不要把它加进 `.gitignore`**；
2. 如果 `Options for Target → Debug` 里显示的是 **ULINK2/ME Cortex-M**（新工程默认值），
   就会报 `No ULINK2/ME Device found` —— 在下拉框改成 **ST-Link Debugger**，
   并在 `Settings → Flash Download` 里勾 `Reset and Run`、
   `Utilities` 里勾 `Use Debug Driver`，一次即可，之后会存回 `.uvoptx`；
3. 用文件名复制工程时，`.uvprojx` 与 `.uvoptx` **必须同名**。

- **本工程不用 CubeMX 生成**：GPIO/定时器初始化都写在 `motor.c` / `servo.c` / `ps2.c`，
  串口在 `stm32f1xx_hal_msp.c`。原 `ps2.ioc` / `.mxproject` 已移到备份目录，请不要再生成
  （会产出一套重复的 `MX_*_Init` 和 MSP，与现有代码打架）。

## 已知限制（故意没改，写清原因）

1. **PS2 没有做 ACK 握手**：协议每字节后有一个 ACK 位，标准做法是等 ACK 再发下一字节。
   本驱动沿用"固定延时 + 裕量"（与 PS2X 库相同，实车能跑）。改它需要逻辑分析仪确认
   ACK 波形，盲改会把能跑的车改坏。若出现偶发 `pad=lost` 或按键整体乱一帧：
   ①把 `ps2.c` 的 `PS2_LOCK_IRQ` 改 `1`；②再用逻辑分析仪确认 ACK 后引入握手。
2. **看门狗只在主循环阶段生效**：`Wdg_Start()` 放在所有初始化成功之后，初始化失败会停在
   `Error_Handler` 里把"文件:行号"打出来。调试器停住时 IWDG 被 `DBGMCU` 冻结，不会咬狗。
3. HAL 的 DMA/FLASH/EXTI/PWR 模块仍开着（CubeMX 默认），实际没用，关掉能省约 1KB Flash——
   属可选优化，没动，避免修改已验证的编译配置。
