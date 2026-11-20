# 电控组第四次培训作业

- **张俊宏**
- ***20263008258***

> 芯片：STM32F103C8T6　库：HAL　环境：STM32CubeMX 6.18.1 + CLion 2026.2 + arm-none-eabi-gcc 15.2.1

---

## 一、任务

用 TIM2_CH1 输出 50Hz PWM 驱动 SG90 舵机，依次转到 0° / 45° / 90° / 135° / 180°，
每点停 1 秒；OLED 显示「目标角度 / CCR / 脉宽」三行，串口同步打印同样内容。

选做（进阶 A）已完成：两个按键每次 ±15°，限幅 0°~180°。

---

## 二、参数推导

### 1. 定时器时钟是 72 MHz

TIM2 挂在 APB1 上。系统主频 72MHz，APB1 最高 36MHz，所以 APB1 分频系数为 2。

但 STM32 有一条规则：**当 APB 分频系数不等于 1 时，定时器时钟会被硬件自动乘 2**。

```
72MHz → APB1 = 36MHz → 定时器时钟 = 36 × 2 = 72MHz
```

所以 PSC 要按 72MHz 来算，不能按 36MHz。

### 2. 让 1 个计数 = 1 µs

```
72MHz ÷ (PSC + 1) = 1MHz
PSC + 1 = 72
PSC = 71
```

这样配好之后，**CCR 的数值直接等于脉宽的微秒数**，调试时一眼能看出对不对，中间不用换算。

### 3. 让周期 = 20 ms（50 Hz）

```
1MHz × 20ms = 20000 个计数
ARR + 1 = 20000
ARR = 19999
```

### 4. 验算

```
72MHz ÷ (71+1) ÷ (19999+1) = 72,000,000 ÷ 72 ÷ 20000 = 50 Hz ✓
```

| 参数 | 值 |
|---|---|
| Prescaler (PSC) | 71 |
| Counter Period (ARR) | 19999 |
| 输出频率 | 50 Hz |
| 1 个计数 | 1 µs |

---

## 三、角度与脉宽

SG90 靠脉宽定角度：0.5 ms 对应 0°，2.5 ms 对应 180°，线性关系。

**两端为什么取 550 / 2450：**

0.5 ms 和 2.5 ms 是名义值。真实的 SG90 行程普遍比标称窄一点，打满会「咔咔」顶到机械限位，堵转发热，时间长了会烧舵机。

所以两端各留约 2° 的余量，把映射区间收成 **2°~178° 对应 550~2450 µs**：

```c
if (angle < 2.0f)   angle = 2.0f;
if (angle > 178.0f) angle = 178.0f;
return (uint16_t)(SERVO_MIN_US
                + (angle - 2.0f) * (SERVO_MAX_US - SERVO_MIN_US) / 176.0f);
```

代进去的结果：

| 角度 | CCR（=脉宽 µs） | 屏上第三行 |
|---|---|---|
| 0° | 550 | Pulse :0.550ms |
| 45° | 1014 | Pulse :1.014ms |
| 90° | 1500 | Pulse :1.500ms |
| 135° | 1986 | Pulse :1.986ms |
| 180° | 2450 | Pulse :2.450ms |

90° 正好落在 1500，可以用这一条快速判断公式写对没有。

---

## 四、接线

| 信号 | 接到 | 说明 |
|---|---|---|
| 舵机 橙线（信号） | PA0 | TIM2_CH1 |
| 舵机 红线（VCC） | **独立 5V 电源正极** | 不接板子的 5V |
| 舵机 棕线（GND） | 独立 5V 电源负极 | **同时与板子 GND 相连** |
| OLED SCL | PB8 | 软件 I²C |
| OLED SDA | PB9 | 软件 I²C |
| OLED VCC | 3.3V | 不能接 5V |
| CH340 TX | PA10 | 交叉接 |
| CH340 RX | PA9 | 交叉接 |
| 按键「+15°」 | PA1 | 输入上拉，另一脚接 GND |
| 按键「−15°」 | PA2 | 输入上拉，另一脚接 GND |

**两条必须照做的地方：**

1. **舵机独立供电。** SG90 堵转电流接近 700mA，从板子的 5V 取电会掉压，轻则抖动，重则单片机反复复位。
2. **必须共地。** 独立电源的 GND 一定要和板子的 GND 连在一起，否则 PA0 输出的信号对方「看不懂」。

---

## 五、程序结构

四个模块只操作一份状态，输出统一走一个函数：

| 文件 | 作用 |
|---|---|
| `Core/Src/main.c` | PWM 启动、定角序列、按键扫描 |
| `Core/Src/Servo.c` | 角度→CCR 换算、设置比较寄存器、OLED 三行显示 |
| `Core/Src/OLED.c` | 软件 I²C OLED 驱动（群文件提供） |

所有改角度的路径最后都调同一个出口，保证舵机、屏幕、串口三处一致：

```c
static void ShowAndSend(float a)
{
  uint16_t ccr = Servo_AngleToCCR(a);
  Servo_SetAngle(a);        /* 舵机 */
  Servo_ShowOnOLED(a, ccr); /* 屏幕 */
  printf("Target=%3d  CCR=%4d  Pulse=%d.%03dms\r\n",
         (int)a, ccr, ccr / 1000, ccr % 1000);   /* 串口 */
}
```

按键扫描（20ms 消抖 + 等松手，防止按住连发）：

```c
if (HAL_GPIO_ReadPin(KEY_UP_GPIO_Port, KEY_UP_Pin) == GPIO_PIN_RESET)
{
  HAL_Delay(20);
  if (HAL_GPIO_ReadPin(KEY_UP_GPIO_Port, KEY_UP_Pin) == GPIO_PIN_RESET)
  {
    angle += 15.0f;
    if (angle > 180.0f) angle = 180.0f;
    ShowAndSend(angle);
    while (HAL_GPIO_ReadPin(KEY_UP_GPIO_Port, KEY_UP_Pin) == GPIO_PIN_RESET) { }
  }
}
```

**串口重定向**（GCC 下是 `_write`，不是 Keil 的 `__io_putchar`）：

```c
int _write(int file, char *ptr, int len)
{
  HAL_UART_Transmit(&huart1, (uint8_t *)ptr, len, 100);
  return len;
}
```

---

## 六、运行现象

上电：舵机转到 90°，屏幕显示三行，串口每秒打印一次（序列模式）或按键后打印（按键模式）。

![串口输出](docs/serial.png)

| 操作 | 现象 |
|---|---|
| 上电 | 舵机到 90°，OLED 显示 `Target:090deg` / `CCR   :1500` / `Pulse :1.500ms` |
| 按 PA1 | +15°，三处同步更新 |
| 按 PA2 | −15° |
| 到 180° / 0° | 停住不再动，两端没有顶限位的声音 |

---

## 七、遇到的问题

**1. TIM2 的 ARR 是默认值 65535，不是 19999**

CubeMX 里只填了 Prescaler，Counter Period 那一栏没改，生成出来是 65535。
算一下频率：72MHz ÷ 72 ÷ 65536 ≈ **15.26 Hz**，离 50Hz 差了三倍多，舵机会抖动、动作迟滞。
把 Counter Period 改成 19999 后正常。

**2. OLED 停在上一份作业的画面上**

代码在调 `OLED_ShowString`，但没调 `OLED_Init()`。
这个驱动是直接写屏的（没有缓冲区），初始化没执行时 PB8/PB9 还是复位后的浮空输入，数据根本发不出去。
屏幕的显存断电才清，所以一直留着旧内容。补上 `OLED_Init(); OLED_Clear();` 后正常。

**3. 新增的 .c 文件没有参与编译**

新版 CubeMX 的 CMake 模板不再用 `file(GLOB)` 扫描源文件，`cmake/stm32cubemx/CMakeLists.txt` 里是一份写死的清单。
把 `OLED.c` / `Servo.c` 拷进 `Core/Src` 后必须手动加到根 `CMakeLists.txt`：

```cmake
target_sources(${CMAKE_PROJECT_NAME} PRIVATE
    Core/Src/OLED.c
    Core/Src/Servo.c
)
```

**4. 粘贴代码时把函数头重复了一遍**

`Servo_AngleToCCR` 外面套了一层同名函数，GCC 把它当作嵌套函数接受了，编译不报错，但外层函数没有 `return`，返回的是栈上的随机值。
CCR 因此变成几万的乱值（远大于 ARR），输出恒定，舵机纹丝不动。
删掉多余的那层函数头后正常。

---

## 八、参考资料与驱动来源

**参考资料**

- 时钟树与 APB1 分频后定时器时钟 ×2：第 2 课
- 定时器 PSC / ARR 配合、基本定时：第 3 课
- PWM 原理与占空比：第 4 课
- 舵机脉宽与角度的关系：第 5 课

**驱动来源**

| 文件 | 来源 |
|---|---|
| `Core/Src/OLED.c`、`Core/Inc/OLED.h`、`Core/Inc/OLED_Font.h` | 电控组资料包 `02_Driver`，软件 I²C 实现，引脚固化为 PB8/PB9 |
| `Core/Src/Servo.c`、`Core/Inc/Servo.h` | 电控组资料包 `03_Code`，两个 TODO 由本人补充实现 |

`Servo.c` 中的 `Servo_AngleToCCR` 两端限幅与映射公式、`Servo_ShowOnOLED` 三行显示函数为本人编写。
