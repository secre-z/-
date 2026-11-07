# 2026 创新基地班软件考核（第一题）AI 对话记录

STM32F407ZGT6 · GPIO 输入输出与 EXTI · STM32CubeMX + HAL · Keil MDK AC5

本文件是完成该题全过程中与 AI 的对话记录，按主题分为六节。**代码全部由本人手写录入工程**，AI 只负责讲解原理、指出问题、给出单个函数的写法。

---

## 工程环境

| 项目 | 内容 |
|---|---|
| 芯片 | STM32F407ZGT6（LQFP144） |
| 工具链 | STM32CubeMX + HAL 库，Keil MDK AC5 V5.06 |
| 系统时钟 | 16 MHz（HSI，未开 PLL） |
| 输出 | LED0 = PF3、LED1 = PF5、蜂鸣器 = PF1、波形输出 = PC3 |
| 输入 | KEY0 = PC0、KEY1 = PC2、KEY2 = PF7、KEY3 = PF8、WK_UP = PF9，全部上升沿触发 + 内部下拉，低电平有效 |
| 中断 | EXTI0 / EXTI2 / EXTI9_5，NVIC 优先级 0；SysTick 优先级 15 |

## 完成情况

| 序号 | 内容 | 状态 |
|---|---|---|
| 基本 1 | KEY0 切换模式 1/2，交替闪 / 同步闪 | 完成 |
| 基本 2 | KEY1 → LED0 亮 + 蜂鸣 2s 自动关 | 完成 |
| 基本 3 | KEY2 → 3s 倒计时，LED1 以 1Hz 闪，结束长鸣 | 完成 |
| 基本 4 | WK_UP 紧急复位 | 完成 |
| 基本 5 | 模式 3/4 软件呼吸灯（不使用定时器） | 完成 |
| 拓展 1 | 模式 6 GPIO 输出方波，占空比 0~100% 可调（不使用定时器） | 完成 |
| 拓展 2 | 模式 5 下 KEY2 多击检测：单击 / 双击 / 三击 | 完成 |
| 拓展 3 | 模式 5 中呼吸不停 + 多击同时成立 | 完成 |

## 说明

- 记录为按主题整理的版本，删除了重复提问、与代码无关的硬件接线试错过程，以及口语化的情绪表达；**技术问答、代码和排查结论保留原样**。
- 所有代码片段都是当时对话中给出的最终版本，本人根据这些片段手写进工程，并非直接粘贴。

---

# 一、读工程与题目分析

## 问题 1：先让 AI 把整个工程读一遍，报出底层配置、隐患和整体架构

**我：** "D:\CubeMxProject\电协比赛"读这里面的东西，你不要动手改代码，只通过对话给出代码解决我的问题

**AI：** 读完了。工程能编译（`Code=3622`，0 error 0 warning），底层 GPIO/EXTI 也都配好了，缺的是应用层，只剩模式 1/2 两个 500ms 阻塞循环。下面按“现状 → 必须确认的电平 → 架构 → 完整代码 → 验证顺序”来说。

### 一、我读到的硬件事实

| 项 | 值 | 出处 |
|---|---|---|
| 芯片 | STM32F407ZGT6 / LQFP144 | `project.ioc:8` |
| 主频 | **HSI 16MHz，PLL 没开** | `main.c:132-148`、`project.ioc:127` |
| Buzzer | PF1，推挽输出，初值 RESET | `main.h:60-61`、`gpio.c:54-61` |
| LED0 / LED1 | PF3 / PF5，推挽输出，初值 RESET | `main.h:62-65` |
| KEY0 / KEY1 | PC0 / PC2，**上升沿 + 下拉** | `gpio.c:69-73` |
| KEY2 / WK_UP | PF7 / PF9，**上升沿 + 下拉** | `gpio.c:63-67` |
| EXTI 优先级 | EXTI0 / EXTI2 / EXTI9_5 全 = 0 | `gpio.c:76-83` |
| SysTick 优先级 | 15（最低） | `stm32f4xx_hal_conf.h:151` |

主频 16MHz 这点很关键：软件延时的循环次数必须按“1µs ≈ 16 周期”折算，照抄网上 168MHz 的例程会慢 10 倍。

### 二、现有代码的 4 个坑

1. **`main.c:78` 和 `:90` 调了两次 `MX_GPIO_Init()`**，第一次还在 `SystemClock_Config()` 之前，删掉第 78 行那次。
2. **EXTI 回调里永远不能出现 `HAL_Delay`**。EXTI 优先级 0、SysTick 优先级 15，SysTick 抢不进来，会死等。你现在用 `soft_delay` 侥幸躲过，但 `soft_delay(20000)` 在 16MHz 下只有约 5.6ms，不是你以为的 20ms。
3. `mode` 只在 ISR 里改，`main.c:102` 的 `switch` 只有 `case 1/2`，模式 3/4/5 全空转；KEY2/WK_UP 触发了 EXTI 也进不了任何逻辑。
4. `MODE_1/MODE_2` 用 `HAL_Delay(500)` 阻塞，换模式要等半秒才生效，正好撞“按键反应迟钝酌情扣分”这条；而且这种流水写法没法叠加告警。

### 三、动手前必须确认的 3 个电平

1. **按键是“按下 = 高”吗？** 配置是上升沿 + 下拉，意味着按键要接 3.3V。万用表量 PC0 对地：按下变 3.3V 就对；按下变 0V 说明是低有效按键，要在 CubeMX 里改成下降沿 + 上拉，并把下面 `KEY_PRESSED_LEVEL` 改成 `GPIO_PIN_RESET`。
2. **蜂鸣器是高电平响还是低电平响？** 量 PF1 在按 KEY1 时的电平变化。
3. **方波从哪个脚出？** 我默认 PA1，代码里两行宏，随便换。

### 四、架构：软件时间片 + 事件 + 输出仲裁

```
EXTI 中断（只置 1 个标志，10 条指令退出）
        ↓
2ms 软件时间片（纯延时循环，1 PWM 周期 = 100 步 × 20µs）
   ├─ 按键消抖 + 单击/双击/三击
   ├─ 各模式动画（闪烁 / 呼吸 / 方波）
   └─ 告警计时（2s / 3s / 长鸣）
        ↓
输出仲裁（告警 > 模式动画）
        ↓
同一套 PWM 计数器写 4 个引脚
```

三个设计决定，正好对上评分点：

- **PWM 周期 2ms（500Hz）、100 步 → 占空比分辨率 1%**，0~100% 无级可调。LED 亮度、呼吸、方波共用这一个计数器，拓展1 的“范围大、步进细”直接满分。
- **中断里绝不消抖**：只 `s_edge[k]++`。消抖、多击判定全在时间片里，所以“按键不丢失、乱按不死机”是结构上保证的。
- **输出仲裁**：模式动画先算“逻辑亮度”，再让告警覆盖。这就是题目说的“状态标志位跳转”。

### 5.3 `Core/Src/main.c` 的 `main()`（替换第 65~113 行）

```c
int main(void)
{
  HAL_Init();
  SystemClock_Config();      /* 先配时钟，再配 GPIO */
  MX_GPIO_Init();            /* 只调一次！删掉原来第 78 行那次 */

  /* USER CODE BEGIN 2 */
  App_Init();
  /* USER CODE END 2 */

  while (1)
  {
    App_Loop();              /* 永不返回：一个 PWM 步 20us */
  }
}
```

`stm32f4xx_it.c`、`gpio.c` 一个字都不用改。

### 六、上机验证顺序（照这个顺序，5 分钟定位问题）

1. **先只验按键**：模式 1 下按 KEY0，LED0/LED1 应该立刻换相，而不是等半秒。4 个键都点一遍、每个键长按 2 秒，确认长按只触发一次。
2. **量方波**：进到模式 6，示波器/逻辑分析仪量 PA1，50% 占空比时周期应 ≈ 2.000ms。偏大就加大 `SLICE_WORK_US`，偏小就减小，一次调 1，调到 2.00ms。
3. **校时**：模式 1 每 500ms 换相；用秒表看 2s 告警、3s 倒计时。整体偏快/偏慢说明第 2 步的周期还没准（所有时间都由它派生）。
4. **呼吸手感**：`T_BREATH_MS` 在 8~12 之间调，2s 一个呼吸周期最像人呼吸。
5. **压力测试**：模式 5 下狂按 KEY2（单击/双击/三击乱序）+ 同时按 KEY1 + 偶尔按 WK_UP，不应该卡死或状态错乱。
6. **一次展示**：全程不按 RESET、不重烧，按 1→2→3→4→5→6 走一遍。

### 七、答辩可以这么说（对应评分点）

- “呼吸灯和方波的亮度/占空比完全由 `sw_delay_us()` 这个软件循环决定，没用 TIM，也没用 SysTick 中断计数，100 步分辨率、0~100% 无级。”
- “EXTI 中断里只置一个标志位就退出，消抖和多击判定全在 2ms 时间片里做，所以按多快都不丢键，也不会有中断里延时的死等问题。”
- “所有输出最后都收敛到 `output_update()` 一个仲裁函数，告警优先级高于动画，所以任何状态都能随时打断、随时切回去。”

两点补充：

- 如果老师严格到“SysTick 也不许开”，在 `HAL_Init()` 后加一行 `HAL_SuspendTick();`；但从此**任何地方都不能再用 `HAL_Delay`**，否则死等。
- Keil 里把 `Mycode.c` 编码设成 UTF-8（Edit → Configuration → Encoding），否则中文注释乱码。验收前记得把这段对话记录导出发给老师（题目要求）。

## 问题 2：只要问题不要整份代码，先看我现在写的 KEY0 + 模式 1、2

**我：** 我要你给我全部代码啊，我要自己写啊，你给我我能学到啥，就看看我现在的代码，有问题的话就指。我现在刚做key0控制mode1，2

**AI：** 我上一轮给的代码太多，这次只挑问题，你自己写。

先说你现在写对的地方，免得改了不该改的：

- `MODE_1` 的时序是对的：LED0 亮 500ms，紧接着 LED1 亮 500ms，中间没有两个灯同时熄灭的空档，也没有重叠。
- `MODE_2` 的“一起亮、一起灭”也对。
- `mode = (mode == 1) ? 2 : 1` 这句对。

### 现在就会暴露的两个问题

**问题 1：按一下 KEY0，大概率什么都不发生，偶尔才切换一次。**

原因在 `Mycode.c:47-55` 的逻辑链。`HAL_GPIO_EXTI_IRQHandler` 是**先清挂起标志、再调回调**。你在回调里 `soft_delay(20000)` 阻塞了约 6ms（16MHz 下 `while(t--)` 一次约 5 周期，20000×5/16M ≈ 6.3ms）。按键抖动集中在头 1~3ms，这几个上升沿在你阻塞期间重新置起 pending，于是你退出 ISR 后**立刻又进一次**：第二次 `soft_delay` 6ms、读引脚仍是高、`key_flag` 又被置 1、`KEY_0` 又切一次。

净效果：抖出偶数个边沿 → 切两次，回到原模式，你看到“按了没反应”；抖出奇数个 → 切一次，看起来正常。所以表现是**时灵时不灵**，而不是稳定失灵，这会骗你很久。

根子是两个：

- 用“延时后读电平”当消抖，但**没有锁存“这次按下已经处理过了”**。必须有一个标志：确认按下后置位，之后所有上升沿都忽略，直到读到低电平（松开）才解锁。
- `key_flag` 完全没起到这个作用，因为它被 `KEY_0()` 立刻清 0 了，等于每次进 ISR 都重新允许切换。

顺带：`key_flag` 这个变量可以整个删掉。现在是“ISR 里置 1 → 调 `KEY_0()` → 里面判断是不是 1 → 再读一次引脚”，同一件事判断了三遍。ISR 里确认按下后直接切 `mode` 就行，`KEY_0()` 这个函数也可以不存在。

**问题 2：按完 KEY0 要等最多 1 秒才看出来。**

`MODE_1` 里两个 `HAL_Delay(500)`。你按下 KEY0 时，`mode` 确实立刻变成了 2，但 CPU 正卡在 `HAL_Delay` 里，必须把当前这 500ms 甚至下一个 500ms 走完，`MODE_2` 才会被调用。最坏情况接近 1 秒。

这就是题目里“按键反应迟钝酌情扣分”那一条，而且它不只影响手感：后面 KEY1 的 2 秒告警、KEY2 的 3 秒倒计时、WK_UP 的紧急复位，全都要求“立刻响应”。只要主循环里还存在 `HAL_Delay`，基本功能 4 的“无论系统处于何种状态，立刻关闭”就永远做不到，因为你没法在 `HAL_Delay` 里响应。

方向只有一个：**MODE_x 函数不再“等待”，而是“判断然后立即返回”**。给每个模式一个状态变量（比如“现在该谁亮”），函数每次进来只做一件事：看时间够不够 500ms，够了就翻转一次并记下翻转时刻，然后立刻 return。这样一次调用只有几微秒，`mode` 一变下一轮就生效。

这个改完，你才具备写后面 4 个功能的地基。现在这个“流水式”写法，题目第三节第 1 条已经明确说了会出问题。

### 迟早会暴露的问题（现在可以先记着）

- `MODE_1`/`MODE_2` 都在**函数末尾顺手把灯关掉**，返回时两个灯必然是灭的，所以现在切换不会残留。一旦你按上面改成“判断后立刻返回”，这个隐式约定就断了：从 `MODE_2` 的“都亮”阶段切走，回到 `MODE_1` 时 LED1 还亮着，就会出现**双灯同亮**、违反“交替”。改成状态机时，每个模式的第一句应该是把不属于自己的灯关掉，而不是依赖上一个函数收尾。
- `mode` 建议加 `volatile`。现在跨文件访问，编译器不会缓存它，所以能跑；但你把 `mode` 挪进 `main.c`、或者开了 LTO、或者改成同文件静态变量，`switch(mode)` 就可能被优化成只读一次，模式永远不切。这是那种“改个位置就莫名其妙坏了”的坑。
- `soft_delay` 的空循环 `while(t--){}` 在高优化等级下可能被改写成更快的循环，延时变短。循环体里放个 `__NOP()` 更可控。另外它现在是 6ms，不是你以为的 20ms，而典型轻触按键抖动 5~20ms，6ms 挡不住。
- **永远不要在 EXTI 回调里用 `HAL_Delay`**。EXTI 优先级是 0（`gpio.c:76`），SysTick 是 15（`stm32f4xx_hal_conf.h:151`），SysTick 抢不进来，`uwTick` 不涨，直接死等。你现在用 `soft_delay` 侥幸躲过，别顺手换成 `HAL_Delay`。
- `main.c:78` 那次 `MX_GPIO_Init()` 在 `SystemClock_Config()` 之前，而且第 90 行又调了一次。删掉第 78 行。

### 一个评分风险

“闪烁周期 500ms”有三种读法：你现在是“每个灯亮 500ms、灭 500ms”（一个明暗循环 1s）；也可能被理解成“一个完整明暗循环 500ms”（亮 250 灭 250）。

最稳的做法是**亮 250ms、灭 250ms，且两个灯相位相反**——这样“周期 500ms”和“交替”两个词同时成立，怎么问都说得通。你现在 500/500 的做法如果被按字面问“周期是多少”，答案会是 1 秒。

先改问题 1 还是问题 2？我建议先改 2（把 `HAL_Delay` 从 `MODE_x` 里拿掉），因为它决定了整个代码结构；问题 1 只是加一个锁存标志，十分钟的事。

你现在按 KEY0 的实际现象是“要等一下才变”，还是“有时候按了没反应”？如果是后者，那就是问题 1 在发作，别去调延时参数，调了也没用。

## 问题 3：只要指出问题，一行实现代码都不要给

**我：** 不要你给全部代码

**AI：** 明白，以后只指问题，不贴实现。哪怕是三行也不给，你自己写。

你现在这一步（KEY0 + 模式1、2）只有两个真问题：

1. `MODE_x` 里的 `HAL_Delay(500)` 让按键最多等 1 秒才生效，并且决定了你没法响应 WK_UP 的“立刻”。
2. `soft_delay` 消抖没有“已处理”锁存，抖动会让同一次按下进两次 ISR，切两次模式等于没切。

其余（volatile、空循环被优化、`MX_GPIO_Init` 调两次）是隐患，不影响你现在跑。

改完把你改后的 `MODE_1` 和 KEY0 那段贴过来，我只说哪里还会出问题。或者你直接问具体某一句为什么这么写不对，我讲原理。

## 问题 4：烧录后按 KEY0 没反应，一直停在 mode1

**我：** 我现在的问题就是烧录后，我控制不了，他一直在mode1

**AI：** “一直在 mode1” 有两种完全不同的原因，导致的现象却一模一样。先用一个实验分开，不然你会白改。

### 判别实验（改一行，5 秒）

把 `KEY_0()` 里那句 `mode = (mode == 1) ? 2 : 1;` 临时改成 `mode = 2;`，烧录，按一下 KEY0。

- **变成模式2 了** → 中断在正常进，问题是“一次按下切了两次”，正反抵消，净效果为 0。往下看 A。
- **还是模式1** → 中断根本没进。往下看 B。

前提：先确认模式1 本身在跑，即两个灯在正常交替。如果在跑，说明主循环和 GPIO 输出都是好的，问题只在按键这条链上。

### A. 为什么“一次按下切两次”是必然的，不是偶尔

`HAL_GPIO_EXTI_IRQHandler` 的顺序是**先清挂起标志、再调回调**。你在回调里 `soft_delay(20000)` 堵了 4~6ms，而按键的抖动边沿全部发生在刚接触的那几毫秒——正好落在你堵住的这段时间里。它们把 EXTI 的挂起位置起来了，你退出 ISR 后 NVIC 立刻再进一次。第二次照样延时、照样读到高电平、照样切一次。

**一次按下，恰好两次切换，mode 从 1 变 2 再变回 1。** 挂起位只有一个 bit，所以再多的抖动边沿也只合并成一次重入，正好两次。这就是“永远停在 mode1”而不是“时灵时不灵”的原因，跟你按多快、按多久都没关系。

改的方向（不给你代码）：消抖不能靠“等一会儿再看”，要靠**锁存**——确认按下后置一个“已处理”标志，此后所有上升沿一律忽略，直到读到引脚为低（真的松开）才允许下一次。这个标志要放在 `mode` 旁边，和 `key_flag` 完全不是一回事：`key_flag` 现在被 `KEY_0()` 立刻清 0，等于每次进 ISR 都重新开门。

### B. 如果中断根本没进

按这个顺序查，第一条最可能：

1. **万用表量 PC0 对地**：按住 KEY0 时是 3.3V 还是 0V？配置是“上升沿 + 下拉”，只有变成 3.3V 才会触发。如果是 0V，说明你的按键是“按下接地”的低有效接法，那配置和代码要一起翻：下拉改上拉、`IT_RISING` 改 `IT_FALLING`、代码里 `GPIO_PIN_SET` 改 `GPIO_PIN_RESET`。只改代码里那个判断是没用的，因为中断压根不会来。
2. **在 `HAL_GPIO_EXTI_Callback` 第一行下断点**，全速跑，按 KEY0。不进断点 → 中断确实没来；进了但立刻又进一次 → 就是 A。
3. 顺便看一眼 `gpio.c`：PC0 时钟已经使能（`__HAL_RCC_GPIOC_CLK_ENABLE`），`EXTI0_IRQn` 也 `HAL_NVIC_EnableIRQ` 了，这两处是对的，不用怀疑。

### 更严谨的判别（顺便练调试）

在回调开头放一个 `volatile uint32_t isr_cnt; isr_cnt++;`，Keil 里把它加进 Watch 窗口，按 10 下 KEY0。读数是 10 → 中断正常、每次一下；是 20 左右 → 就是 A 的双重进入。这个数比“看灯”可靠得多，以后调多击检测也用得上。

跑完告诉我哪边，我再往下指。别急着改延时参数，那个 `20000` 无论调大调小都治不了这个病。

---

# 二、按键与 EXTI

## 问题 1：烧录后怎么按都停在模式 1

**我：** 我现在的问题就是烧录后，我控制不了，他一直在mode1

**AI：**

“一直在 mode1” 有两种完全不同的原因，导致的现象却一模一样。先用一个实验分开，不然你会白改。

### 判别实验（改一行，5 秒）

把 `KEY_0()` 里那句 `mode = (mode == 1) ? 2 : 1;` 临时改成 `mode = 2;`，烧录，按一下 KEY0。

- **变成模式2 了** → 中断在正常进，问题是“一次按下切了两次”，正反抵消，净效果为 0。往下看 A。
- **还是模式1** → 中断根本没进。往下看 B。

前提：先确认模式1 本身在跑，即两个灯在正常交替。如果在跑，说明主循环和 GPIO 输出都是好的，问题只在按键这条链上。

### A. 为什么“一次按下切两次”是必然的，不是偶尔

`HAL_GPIO_EXTI_IRQHandler` 的顺序是**先清挂起标志、再调回调**。你在回调里 `soft_delay(20000)` 堵了 4~6ms，而按键的抖动边沿全部发生在刚接触的那几毫秒——正好落在你堵住的这段时间里。它们把 EXTI 的挂起位置起来了，你退出 ISR 后 NVIC 立刻再进一次。第二次照样延时、照样读到高电平、照样切一次。

**一次按下，恰好两次切换，mode 从 1 变 2 再变回 1。** 挂起位只有一个 bit，所以再多的抖动边沿也只合并成一次重入，正好两次。这就是“永远停在 mode1”而不是“时灵时不灵”的原因，跟你按多快、按多久都没关系。

改的方向（不给你代码）：消抖不能靠“等一会儿再看”，要靠**锁存**——确认按下后置一个“已处理”标志，此后所有上升沿一律忽略，直到读到引脚为低（真的松开）才允许下一次。这个标志要放在 `mode` 旁边，和 `key_flag` 完全不是一回事：`key_flag` 现在被 `KEY_0()` 立刻清 0，等于每次进 ISR 都重新开门。

### B. 如果中断根本没进

按这个顺序查，第一条最可能：

1. **万用表量 PC0 对地**：按住 KEY0 时是 3.3V 还是 0V？配置是“上升沿 + 下拉”，只有变成 3.3V 才会触发。如果是 0V，说明你的按键是“按下接地”的低有效接法，那配置和代码要一起翻：下拉改上拉、`IT_RISING` 改 `IT_FALLING`、代码里 `GPIO_PIN_SET` 改 `GPIO_PIN_RESET`。只改代码里那个判断是没用的，因为中断压根不会来。
2. **在 `HAL_GPIO_EXTI_Callback` 第一行下断点**，全速跑，按 KEY0。不进断点 → 中断确实没来；进了但立刻又进一次 → 就是 A。
3. 顺便看一眼 `gpio.c`：PC0 时钟已经使能（`__HAL_RCC_GPIOC_CLK_ENABLE`），`EXTI0_IRQn` 也 `HAL_NVIC_EnableIRQ` 了，这两处是对的，不用怀疑。

跑完告诉我哪边，我再往下指。别急着改延时参数，那个 `20000` 无论调大调小都治不了这个病。

## 问题 2：让 AI 直接看代码定位，问题锁定在一处

**我：** 你去我文件里面看我的代码，然后指出哪里有问题，然后给出解决问题的代码，不是什么都不给，你再看看，我改了还是不行

**AI：**

看完你现在的代码了（`Mycode.c` 57 行、`gpio.c` 89 行、`main.c` 189 行）。

### 先说结论：你这一改，把嫌疑范围缩到只剩一处

`Mycode.c:39` 现在是写死的 `mode = 2;`。**只要这行被执行过一次，mode 就永远是 2，主循环必进 `MODE_2`（两灯同亮同灭）。** 你说还是模式1，那就只有一个解释：

> `Mycode.c:50` 那个 `if` 从来没成立过，所以第 52~53 行一次都没跑。

能进到第 50 行，说明中断是进来的（第 49 行延时都执行了）。于是问题就锁定在：**`HAL_GPIO_ReadPin(GPIOC, Key0_Pin) == GPIO_PIN_SET` 读回来的不是高电平。**

而这里有个物理上很硬的推论：上升沿意味着那一刻 PC0 确实是高的，5ms 后你手还按着，它应该还是高的。它却不是——说明**让你进中断的那个“上升沿”根本不是按键按下的沿**（干扰、上电毛刺），或者**松手太快**，更多情况是：**你的按键根本不是“按下接 3.3V”这种接法**。

`gpio.c:71-72` 给 PC0 配的是 `GPIO_MODE_IT_RISING + GPIO_PULLDOWN`，即“空闲被拉到低、按下变高”。如果实物是“按下接地”（绝大多数开发板都是这样），那么按下是 0V，既不会来上升沿，读回来也永远是低——你现在的现象 100% 吻合。只改代码里那个 `GPIO_PIN_SET` 是没用的，因为中断压根不会来。

不用写代码解决，把 `gpio.c:71-72` 的 `GPIO_PULLDOWN` 改成 `GPIO_PULLUP`、`GPIO_MODE_IT_RISING` 改成 `GPIO_MODE_IT_FALLING`——前提是你板子上 KEY0 按下确实是接地。

**板子原理图上 KEY0 那一头接的是 3.3V 还是 GND？** 你看一眼告诉我，或者直接万用表量：按住时 PC0 对地是 3.3V 还是 0V。这个答案一出来，剩下的就不用猜了。

## 问题 3：不改 CubeMX 配置，坚持用中断

**我：** 就按我的逻辑来，不要让我改cubemx里的配置啊（有问题另说），我现在是中断，就用中断

**AI：**

行，配置一行不动，还是 `GPIO_MODE_IT_RISING + GPIO_PULLDOWN`，还是中断。只把中断服务函数里那两句换掉。

### 为什么只换这两句

`Mycode.c:49-50` 现在是“延时 5ms 再看电平”。上升沿已经说明按下这件事发生了，再读一次电平是第二重验证——**而这一重验证正是你这次卡住的地方**。新版把它删掉，改成“记账 + 锁存”：

- 沿来了 → 立刻记一次，置 `key_lock`；
- `key_lock` 为 1 时再来多少沿都当放屁（抖动重入被吃掉）；
- 主循环看到脚回到低（松手）才把 `key_lock` 清 0。

这样中断里一个 `while` 都没有，也不会重入两次。**只要有一个上升沿，模式就一定会切**——比你现在少了一个可能失败的环节。

### Mycode.c 全文替换

```c
#include "Mycode.h"
#include "main.h"

uint8_t  mode = 1;
static volatile uint8_t key_flag = 0;   /* 1 = 有一次按下待处理 */
static volatile uint8_t key_lock = 0;   /* 1 = 这次按下已记账，松手才清 0 */

volatile uint32_t key0_irq_cnt = 0;     /* 调试用：数一下进了几次中断，正式版可删 */

static uint32_t blink_tick = 0;
static uint8_t  phase = 0;              /* 0:LED0 亮  1:LED1 亮 */

/* ---------------- 中断：只记账，不延时、不读电平 ---------------- */
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == Key0_Pin)
    {
        key0_irq_cnt++;

        if (key_lock == 0)              /* 锁住期间所有抖动沿一律丢弃 */
        {
            key_lock = 1;
            key_flag = 1;
        }
    }
}

/* ---------------- 主循环：处理按键 ---------------- */
void KEY_0(void)
{
    if (key_flag)
    {
        key_flag = 0;
        mode = (mode == 1) ? 2 : 1;     /* 你的逻辑，原样保留 */
        phase = 0;
        blink_tick = HAL_GetTick();
    }
}

/* ---------------- 主循环：一个 tick 干完所有事 ---------------- */
void App_Tick(void)
{
    uint8_t l0, l1;

    /* 松手解锁：这就是消抖，"等"这个动作搬到了主循环，不在中断里 */
    if (key_lock && HAL_GPIO_ReadPin(GPIOC, Key0_Pin) == GPIO_PIN_RESET)
    {
        key_lock = 0;
    }

    KEY_0();

    if (HAL_GetTick() - blink_tick >= 500u)     /* 到点才翻，绝不等待 */
    {
        blink_tick += 500u;
        phase ^= 1u;
    }

    if (mode == 1) { l0 = (phase == 0u); l1 = (phase != 0u); }
    else           { l0 = (phase == 0u); l1 = l0; }

    HAL_GPIO_WritePin(GPIOF, LED0_Pin, l0 ? GPIO_PIN_SET : GPIO_PIN_RESET);
    HAL_GPIO_WritePin(GPIOF, LED1_Pin, l1 ? GPIO_PIN_SET : GPIO_PIN_RESET);
}
```

`Mycode.h` 改成：

```c
#ifndef __MYCODE_H
#define __MYCODE_H

#include "main.h"

void App_Tick(void);
extern uint8_t mode;
extern volatile uint32_t key0_irq_cnt;

#endif
```

`main.c`：删掉 78 行那句 `MX_GPIO_Init();`（它在 `SystemClock_Config` 前面，重复了，90 行那句才是要的），主循环 97-111 行换成：

```c
  while (1)
  {
    App_Tick();
  }
```

### 如果还是不动，先看这个数

`key0_irq_cnt` 就是给你留的探针，不用改任何配置。Keil 里：

1. `Ctrl+F5` 进调试；
2. `View → Periodic Window Update` 勾上（运行中才刷新）；
3. `Watch 1` 窗口里输入 `key0_irq_cnt`；
4. `F5` 全速跑，按几下 KEY0。

三种读数对应的结论：

- **一直是 0** → PC0 上一个上升沿都没来。你的配置、代码都没问题，是那个脚上根本没有“按下变高”这回事。到这一步就不是写代码能解决的了，我们再回头看原理图/接线（这就是你说的“有问题另说”）。
- **每按一次 +2、+4、+6** → 抖动的沿确实在重入，但锁存把它们吃掉了，`mode` 只翻一次。这正好验证了锁存是对的。
- **每按一次 +1，但模式不变** → 那才是代码问题，把你看到的数告诉我。

## 问题 4：只要两个函数，别给长段代码

**我：** 你现在给的这些……（略），给一点简单的函数不行吗，你只告诉我改哪一段，我现在要的只是：mode1函数进入模式1，mode2函数进入模式2，然后我按key0可以控制mode1，2就这样，……（略）

**AI：**

就按你的结构来，你的 `MODE_1`、`MODE_2` 一个字都不用动。只改下面四处，其余全部不动。

**① `Mycode.c:3-4`，加两个 `volatile`**

```c
volatile uint8_t mode = 1;
volatile uint8_t key_flag = 0;
```

**② `Mycode.c:45-56`，中断函数整个换成这样**（就四行逻辑）

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
	if(GPIO_Pin == Key0_Pin)
	{
		if(key_flag == 0)              /* 还没处理完就别理后面的抖动 */
		{
			key_flag = 1;
			mode = (mode == 1) ? 2 : 1;
		}
	}
}
```

`soft_delay` 和那个 `ReadPin==SET` 都删掉。上升沿本身就是“按下”的证据，不用再读一次。

**③ `Mycode.c:33-43`，`KEY_0()` 直接删掉**（逻辑搬进中断了，留着也没人调）

**④ `main.c:101`，`while(1)` 里加一行**，加在 `switch` 前面

```c
	  if(key_flag && HAL_GPIO_ReadPin(GPIOC,Key0_Pin)==GPIO_PIN_RESET)
		  key_flag = 0;                 /* 松手才解锁，下次按下才算数 */
```

`Mycode.h` 里 `KEY_0` 的声明和 `soft_delay` 的声明一起删，`mode`/`key_flag` 前面加 `volatile`。

### 这版的行为

- 按下 KEY0 → 中断里立刻改 `mode`，**按住不放只切一次**（`key_flag` 锁着）；
- 松手后 `main` 里那行把 `key_flag` 清 0，下一次按下又生效；
- 一次按下最多切一次，不会正反抵消。

### 这版的两个已知短板（先能跑，别再改）

1. **响应慢**：`MODE_1`/`MODE_2` 里是 `HAL_Delay(500)`，你按下时它正在等，最多 1 秒才切过去。能用，但“响应快”那条会扣分。
2. **连按快于 1 秒会丢一次**：`key_flag` 要到主循环转回来才清。

这两条是同一个病根——`HAL_Delay`。等你确认按键能切了，再把 `MODE_1` 里的 `HAL_Delay` 换成“记时间戳比较”，一次解决。先烧这版，看按了到底切不切。

## 问题 5：按键是一边 IO、一边接 VCC 吗

**我：** 还是没反应，按键是一边io口，一边接vcc吗

**AI：**

不一定。按键有两种接法，你的配置假设的是**接 VCC**（按下变高、松开变低）。另一种更常见：**接 GND**（按下变低），很多开发板都是这种。接错了，中断永远不来。

不用猜，也不用万用表，写 4 行就能看出来。把 `main.c` 的 `while(1)` 临时换成：

```c
  while (1)
  {
    HAL_GPIO_WritePin(GPIOF, LED0_Pin, HAL_GPIO_ReadPin(GPIOC, Key0_Pin));
  }
```

LED0 会实时显示 PC0 的高低。按住 KEY0 看灯：

- **灯亮** → 按下是高电平，你配置是对的。那中断应该能进，是别的问题，把现象告诉我。
- **灯不亮（一直灭）** → PC0 根本没被拉高。要么按键接的是 GND，要么这个按键压根不在 PC0 上。

如果是接 GND，改三处（可以只改代码，不用重新生成 CubeMX）：

1. `gpio.c:72` 的 `GPIO_PULLDOWN` → `GPIO_PULLUP`
2. `gpio.c:71` 的 `GPIO_MODE_IT_RISING` → `GPIO_MODE_IT_FALLING`
3. 你中断里那句 `mode = (mode==1)?2:1;` 不用改，判断松手那句的 `GPIO_PIN_RESET` 改 `GPIO_PIN_SET`

先烧上面那 4 行，告诉我灯亮不亮。这一个信息就能定死。

## 问题 6：为什么“要狂点按键才能切换”

**我：** 你再看看我的代码，还是不行，我要狂点按键才能切换

**AI：**

看到原因了，跟抖动没关系——**是你那把“锁”根本没人开**。

`main.c:101` 那行解锁代码，只在主循环转回来时执行一次；可主循环 99% 的时间都卡在 `MODE_1`/`MODE_2` 的 `HAL_Delay(500)` 里。也就是说，**每 1 秒才轮到它执行一次**。所以按下一次后，`key_flag` 会一直挂着将近 1 秒，这期间中断里 `if(key_flag == 0)` 把所有按键全挡掉。你狂点，总有一次碰巧落在解锁之后——这就是“狂点才能切”的全部原因。

### 改法：锁用“时间窗”释放，不用“松手”释放

抖动只发生在几毫秒内，那就把 **150ms 内的沿都当成同一次按下**，超过 150ms 才算新的一次。锁不再依赖主循环，中断自己就能判断。

**① `Mycode.c:3-4`**

```c
volatile uint8_t mode = 1;
```

`key_flag` 整个删掉（下面两处会删）。

**② `Mycode.c:35-46`，中断整个换成**

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
	static uint32_t last = 0;
	uint32_t now = HAL_GetTick();

	if(GPIO_Pin == Key0_Pin)
	{
		if(now - last > 150)          /* 150ms 内的沿都算同一次按下的抖动 */
		{
			last = now;
			mode = (mode == 1) ? 2 : 1;
		}
	}
}
```

`soft_delay(500)` 也删掉，它没能起到消抖作用，而且按现在的写法它只有 0.1ms。

**③ `main.c:101-102` 那两行删掉**，`switch` 前面什么都不留。

**④ `Mycode.h`** 里删掉 `soft_delay` 和 `key_flag` 的声明，`extern uint8_t mode;` 改成 `extern volatile uint8_t mode;`。

### 改完会怎样

按一下切一次，按住不放不重复切，按多快都跟得上——因为判断全在中断里，不靠主循环配合。

**但还是有个滞后**：模式是在中断里立刻变的，可 `MODE_1` 里那句 `HAL_Delay(500)` 不会被打断，得等它走完才轮到 `switch` 重新选。所以按键切了之后，灯最多要等 1 秒才变。这不是 bug，是 `HAL_Delay` 的固有代价，先忍，等你确认切换利索了，再把 `MODE_1`/`MODE_2` 的 `HAL_Delay` 换成“查时间戳”，滞后立刻消失。

## 问题 7：把 MODE 函数里的延时改成非阻塞

**我：** 还是不行，到底是为什么啊，……（略），如果是mODE函数里的ddelay函数的问题就改啊，……（略）

**AI：**

**把你越按越没反应的原因说清楚**：中断没问题，`mode` 确实在中断里变了。但 `MODE_1` 里的 `HAL_Delay(500)` 堵着，主循环要等它走完才回来看一眼 `mode`；而在这 1 秒里，你每按一下，`mode` 就翻一次——按两下就翻回去了。你越急按得越快，越像没反应。**不是按键不灵，是你按的次数被“攒”成偶数，自己抵消了。**

### 只改这两个函数，其它一行都别动

在 `Mycode.c` 第 10 行（`void MODE_1()` 上面）加两个变量，然后 `MODE_1`、`MODE_2` 整个换成：

```c
static uint32_t tick  = 0;      /* 上次翻转的时刻 */
static uint8_t  phase = 0;      /* 0:甲相位  1:乙相位 */

void MODE_1()
{
	if(HAL_GetTick() - tick >= 500)   /* 到点了才翻，不等待 */
	{
		tick += 500;
		phase ^= 1;
	}
	HAL_GPIO_WritePin(GPIOF,LED0_Pin, phase ? GPIO_PIN_RESET : GPIO_PIN_SET);
	HAL_GPIO_WritePin(GPIOF,LED1_Pin, phase ? GPIO_PIN_SET   : GPIO_PIN_RESET);
}

void MODE_2()
{
	if(HAL_GetTick() - tick >= 500)
	{
		tick += 500;
		phase ^= 1;
	}
	HAL_GPIO_WritePin(GPIOF,LED0_Pin, phase ? GPIO_PIN_RESET : GPIO_PIN_SET);
	HAL_GPIO_WritePin(GPIOF,LED1_Pin, phase ? GPIO_PIN_RESET : GPIO_PIN_SET);
}
```

`main.c`、`Mycode.h`、中断函数都不用动。

### 改完是这样的

这两个函数从“等 1 秒”变成“看一眼就返回”，主循环转一圈只要几微秒，所以 `switch(mode)` 每次都能看到中断刚改的值。**按下键的那一刻，下一个循环就换灯**，不再有 1 秒滞后，也不会攒着抵消。

- 按一下 → 图案立刻换；
- 按住不放 → 只切一次；
- 连按十下 → 图案来回跳十次，不会“看起来没动”。

---

# 三、模式框架与软件呼吸灯

## 问题 1：五个 MODE、三个 able、四个 key 的框架怎么分工才不冲突

**我：** 换种说法，我来说说我的思路，框架：五个MODE_X函数：分别是五种模式下灯的状态。三个able_X函数分别是基本功能2，3，4。四个key_X函数，key0里面就是mode1,2，key1就是able2，key2就是mode5，然后在选择able234，key3(Wk_UP)就是able4。方波先别管，目前就这样，看看怎么实现。你给出每个函数如何实现和按键如何不冲突

**AI：** 这个框架是对的，能撑起整道题。缺的只是三条“纪律”，加上把 `key2` 的一个二义性和 `mode3/4` 的入口补上。

### 一、冲突只有三种，规则定死就不冲突

| 冲突 | 表现 | 规则 |
|---|---|---|
| **事件冲突** | 两键同时按、连击、抖动 | 中断只置位 `ev`，**不在中断里做任何判断**；主循环一个 `dispatch()` 统一按优先级取 |
| **输出冲突** | `MODE_3` 在呼吸，`able_2` 要 LED0 亮 | 所有函数**只写三个缓冲变量** `L0/L1/BZ`，最后由 `flush()` 一次性写脚；谁写谁覆盖，顺序就是优先级 |
| **时间冲突** | 呼吸阻塞 1 秒，able 的 2s 计时就废了 | 每个 `MODE_x` 每调用一次只推进 **≤20ms**，然后立刻 `return` |

优先级就一句：**`able_4`（紧急复位）> 正在进行的 able > MODE 显示**。`able_4` 永远第一个判断，所以任何时候按 WK_UP 都立刻生效。

### 二、你的框架要补两处

1. **`key_0` 只切 1、2 的话，`mode3`/`mode4` 永远进不去**（基本5 的 8 分直接丢）。建议改成 `mode = mode % 4 + 1`，1→2→3→4→1。
2. **`key_2` 有两个用途**：不在 mode5 时“进 mode5”，在 mode5 里“多击”。用 `if (mode == 5)` 区分就行，不冲突，但答辩时要说明这条设计。

### 三、每个函数怎么写

| 函数 | 谁调 | 每次调用做一件什么 | 写哪些 |
|---|---|---|---|
| `MODE_1` | 主循环 | 满 500ms 翻相 | L0=!ph, L1=ph |
| `MODE_2` | 主循环 | 满 500ms 翻相 | L0=L1=!ph |
| `MODE_3` | 主循环 | 占空比走一档 + 软件 PWM 5ms | pwm1=1（LED1 呼吸），L0=0 |
| `MODE_4` | 主循环 | 同上 | pwm0=1，L1=0 |
| `MODE_5` | 主循环 | 同上（呼吸不停） | pwm0=1，L1 交给 able |
| `able_2` | key1 / key2 单击 | 起计时 +2000 | 置 `act=1, act_t0=now` |
| `able_3` | key2 双击 | 起计时 +3000 | `act=2, act_t0=now` |
| `able_4` | key3 / key2 三击 | 一次做完 | `act=0; mode=1; ph=0;` 全关 |
| `key_0..key_3` | `dispatch()` | 只改 `mode` / `act`，**不碰灯** | — |

`act`（当前动作）就三个值：`0` 空闲、`1` able2 进行中、`2` able3 倒计时、`3` able3 长鸣。所有“鸣叫中再按忽略”就是一句 `if (act) return;`。

## 问题 2：哪里可以用 delay，哪里会和中断打架

**我：** 先说补充，新增key5控制进入mode34，方式同key0进入mode12，key2不做基础部分，只做拓展部分（这俩本身就冲突，选了拓展就不做基础的key2了）。然后你不要直接给我这么一长串所谓的“可复制代码”，一时半会看不懂，通过文字+代码的方式分别给出这些mode，key,able代码怎具体么写，怎么实现，主要就是哪里可以用delay，哪里和中断不冲突我不清楚，你先给函数模块，在讲连接

**AI：** 先把你的疑问单独回答，这是整套东西的根：**delay 能不能用，只看它在哪一层。**

| 位置 | 能不能用 delay | 原因 |
|---|---|---|
| EXTI 回调里 | **绝对不行** | 中断里延时会让系统停住，而且抖动沿会在延时期间把挂起位重新置起，退出后马上再进一次；这就是你最早“一次按下切两次”的根源。另外 SysTick 优先级是 15，全系统最低，`HAL_Delay` 在 EXTI 里会**死等**。 |
| `MODE_1`/`MODE_2` 闪烁 | **不要用**，用时间戳 | 这里若阻塞，`able` 的 2s 到点判断会跟着晚；而且按键的响应也要等它走完。 |
| `MODE_3`/`4`/`5` 呼吸 | **必须用**（题目就这么要求） | 但一次只允许 5ms，走完立刻 `return`，下一轮再进来接着走。 |
| `able_2`/`able_3` | **不能用** | 2000ms/3000ms 到点只能靠“查时间戳”，一阻塞就没法同时干别的。 |
| 消抖、判断松手 | **不能用** | 松手只能靠“每轮读一次电平”，不能等。 |
| `dispatch()` | **不能用** | 它只负责“下命令”，一行耗时都不能有。 |

**为什么主循环里阻塞不丢按键**：按键在中断里已经记账了（`ev` 置位），你阻塞 5ms 只是把“处理”推迟 5ms，事件还在。这就是“中断记账 + 主循环执行”的全部意义——**中断不做事，主循环不等人**。

**但阻塞有代价**：`able` 到没到 2 秒，只有在 tick 里才检查得到。你在 `MODE_x` 里堵 1 秒，判定的迟到时间就是 1 秒。所以规则是：`MODE_x` 里阻塞要短（≤20ms），且阻塞期间不能指望别的逻辑推进。

### 一、三层分工

```
EXTI  ──置位──►  ev 位掩码
                     │
              dispatch()       ← 只改 mode / act / clicks，不碰灯，不等待
                     │
     L0=L1=BZ=0; pwm0=pwm1=0
                     │
        ┌────────────┴────────────┐
   act != 0 → action()      act == 0 → render()
   （报警动作优先）          （按 mode 显示）
        └────────────┬────────────┘
                     │
                  flush()        ← 全工程唯一写 GPIO 的地方
```

一句话：**中断只记账，dispatch 只改状态，render/action 只算，flush 只写**。四层各干各的，就不可能打架。

### 二、MODE 层（只算 L0/L1，不写脚）

**`MODE_1`（交替闪）**

```c
void MODE_1(void)
{
	if (HAL_GetTick() - ph_t >= 500) { ph_t += 500; ph ^= 1; }
	L0 = !ph;  L1 = ph;
}
```

要点：写 `ph_t += 500` 而不是 `ph_t = HAL_GetTick()`。后者每轮都重置，周期会被主循环拉长。`ph` 是全局相位，`MODE_1`/`MODE_2` 共用，切换时灯不会跳。

**`MODE_2`（同步闪）**

```c
void MODE_2(void)
{
	if (HAL_GetTick() - ph_t >= 500) { ph_t += 500; ph ^= 1; }
	L0 = L1 = !ph;
}
```

**`MODE_3` / `MODE_4`（呼吸）**

```c
void MODE_3(void) { L0 = 0; pwm1 = 1; }   /* LED1 呼吸，LED0 灭 */
void MODE_4(void) { L1 = 0; pwm0 = 1; }
```

要点：MODE 函数**不发 PWM**，只发“这盏灯这轮走呼吸通道”这个命令。真正的 PWM 在 `flush()` 里跑 5ms。这样呼吸就变成了“每轮 5ms”，不会把 tick 卡死。

**`MODE_5`（呼吸 + 允许报警叠加）**

```c
void MODE_5(void) { pwm0 = 1; L1 = 0; }
```

和 `MODE_3` 唯一的区别是：在 mode5 里 `action()` **允许**覆盖 `L1` 和 `BZ`，但不许碰 `L0`——这就是拓展3“呼吸不停、多击同时成立”的答案。

### 三、KEY 层（两半：中断置位 + dispatch 改名）

**中断（只有这一段，四个键共用）**

```c
void HAL_GPIO_EXTI_Callback(uint16_t pin)
{
	static uint32_t t0, t1, t2, t3, t5;
	uint32_t now = HAL_GetTick();

	if (pin == Key0_Pin  && now - t0 > 150) { t0 = now; ev |= K0; }
	if (pin == Key1_Pin  && now - t1 > 150) { t1 = now; ev |= K1; }
	if (pin == Key2_Pin  && now - t2 > 150) { t2 = now; ev |= K2; }
	if (pin == WK_UP_Pin && now - t3 > 150) { t3 = now; ev |= K3; }
	if (pin == Key5_Pin  && now - t5 > 150) { t5 = now; ev |= K5; }
}
```

每个键一个 `static` 时间戳，互不影响。`150` 就是消抖窗口——同一次按下的所有抖动沿都落在这 150ms 里，只认第一个。

**dispatch（每个 key 一个分支）**

```c
static void dispatch(void)
{
	uint8_t  e = ev;  ev &= ~e;          /* 取走，别用 ev = 0 的方式硬清 */
	uint32_t now = HAL_GetTick();

	if (e & K3) able_4();                /* WK_UP 最高优先级，永远响应 */

	if (e & K0) { mode = (mode == 2) ? 1 : 2; ph = 0; }   /* key0：1、2 */
	if (e & K5) { mode = (mode == 4) ? 3 : 4; ph = 0; }   /* key5：3、4 */

	if ((e & K1) && !act) able_2();      /* key1：报警中忽略 */
	if ((e & K2) && !act)
	{
		if (mode == 5) { clicks++; click_t = now; }       /* 拓展：数连击 */
		else           { able_3(); }                       /* 基本3：直接倒计时 */
	}

	if (clicks && now - click_t >= 250)  /* 窗口到期才分派 */
	{
		uint8_t n = clicks; clicks = 0;
		if (act) return;
		if (n == 1) able_2(); else if (n == 2) able_3(); else able_4();
	}
}
```

## 问题 3：函数里只有一个变量赋值，灯到底在哪亮

**我：** 先别管key5了，你这函数写的啥？一个变量不声明，一个HAL库的函数都没有全是c语言逻辑，你灯怎么亮？

**AI：** 是我讲散了，分节讲的时候把声明和写脚那两块放前面去了，你单看 MODE 函数当然像“只有 C 逻辑、没有 HAL”。不是漏写，是**灯故意不在 MODE 里亮**。

### 灯在哪亮

```c
MODE_1():  L0 = 1;                                        ← 只改一个变量
                    ↓
flush():   HAL_GPIO_WritePin(GPIOF, LED0_Pin,
                             L0 ? GPIO_PIN_SET : GPIO_PIN_RESET);   ← 灯在这一行亮
```

全工程**只有 `flush()` 一处调用 `HAL_GPIO_WritePin`**。这样 `able_2` 想点亮 LED0，改 `L0` 就行，不用知道引脚叫什么；`MODE_x` 也一样。如果每个函数都自己写脚，`able` 和 `MODE` 就会抢同一个脚，后写的覆盖先写的，你又要开始排顺序。

### 声明（文件最上面，一次写全）

```c
static uint8_t  L0, L1, BZ;      /* 本轮期望输出：灯0、灯1、蜂鸣 */
static uint8_t  pwm0, pwm1;      /* 1 = 这盏灯本轮走呼吸通道 */
static uint8_t  ph;              /* 闪烁相位 */
static uint32_t ph_t;            /* 上次翻相位的时刻 */
static uint8_t  act;             /* 0空闲 1 able2 2 able3倒计时 3 长鸣 */
static uint32_t act_t0;
```

### flush

```c
static void flush(void)
{
	if (pwm0) breath(LED0_Pin);
	else HAL_GPIO_WritePin(GPIOF, LED0_Pin, L0 ? GPIO_PIN_SET : GPIO_PIN_RESET);

	if (pwm1) breath(LED1_Pin);
	else HAL_GPIO_WritePin(GPIOF, LED1_Pin, L1 ? GPIO_PIN_SET : GPIO_PIN_RESET);

	HAL_GPIO_WritePin(GPIOF, Buzzer_Pin, BZ ? GPIO_PIN_SET : GPIO_PIN_RESET);
}
```

`breath()` 里也是 HAL 函数，就是先拉高、`soft_delay`、再拉低，只是我把它单独拎出来了。

### 一轮 tick 到底发生了什么

```c
void App_Tick(void)
{
	dispatch();                      /* 改 mode / act */

	L0 = L1 = BZ = 0;                /* 清零 = 本轮默认全灭 */
	pwm0 = pwm1 = 0;

	if (act) action(); else render();/* 报警优先，否则按 mode 显示 */
	flush();                         /* ← 灯在这里统一亮 */
}
```

一轮几微秒，主循环一秒钟跑几十万轮，所以灯是**每轮被重新写一次**的。闪烁不是“亮 500ms 再灭 500ms”，而是“每轮都写一次，而 `L0` 的值在 500ms 的边界上才翻”。这就是不需要 delay 的原因。

## 问题 4：先只要一个 MODE_1 函数

**我：** ……（略）我就这么说，你现在就写一个一个函数，一个mode1的函数行不行

**AI：**

```c
void MODE_1(void)
{
	static uint32_t tick = 0;
	static uint8_t  ph   = 0;

	if (HAL_GetTick() - tick >= 500)
	{
		tick += 500;
		ph ^= 1;
	}

	if (ph == 0)
	{
		HAL_GPIO_WritePin(GPIOF, LED0_Pin, GPIO_PIN_SET);
		HAL_GPIO_WritePin(GPIOF, LED1_Pin, GPIO_PIN_RESET);
	}
	else
	{
		HAL_GPIO_WritePin(GPIOF, LED0_Pin, GPIO_PIN_RESET);
		HAL_GPIO_WritePin(GPIOF, LED1_Pin, GPIO_PIN_SET);
	}
}
```

就这一个函数，直接替你现在的 `MODE_1`，别的什么都不用改。

- `tick`、`ph` 声明在函数里面，不占全局，不需要 `Mycode.h` 配合；
- 到 500ms 翻一次 `ph`，`ph=0` 时 LED0 亮、LED1 灭，`ph=1` 反过来 → 交替闪；
- 函数不阻塞，进去看一眼就出来，主循环一轮一轮地喊它，灯就一直闪。

## 问题 5：MODE_3 的软件呼吸灯怎么写

**我：** 这不就好了吗，我要的就是这种方式告诉我，我顺便把mode2也写了。下面写mode3

**AI：** 模式3：LED0 呼吸，LED1 灭。呼吸只能用软件延时，所以这个函数每轮自己跑一小段 PWM。

```c
void MODE_3(void)
{
	static uint16_t duty = 0;      /* 当前亮度 0~100 */
	static int8_t   dir  = 1;      /* 1 变亮，-1 变暗 */
	static uint32_t step = 0;

	HAL_GPIO_WritePin(GPIOF, LED1_Pin, GPIO_PIN_RESET);

	if (HAL_GetTick() - step >= 10)     /* 每 10ms 亮度走一档 */
	{
		step += 10;
		duty += dir;
		if (duty >= 100) { duty = 100; dir = -1; }
		if (duty == 0)   { dir = 1; }
	}

	/* 软件 PWM：周期 10ms，亮 duty%，灭剩下的 */
	HAL_GPIO_WritePin(GPIOF, LED0_Pin, GPIO_PIN_SET);
	soft_delay((uint32_t)duty * 400);
	HAL_GPIO_WritePin(GPIOF, LED0_Pin, GPIO_PIN_RESET);
	soft_delay((uint32_t)(100 - duty) * 400);
}
```

三个变量放在函数里面，不会和 `MODE_1`、`MODE_2` 用的 `tick`、`phase` 冲突。

- **`400` 这个数怎么来的**：你之前 `soft_delay(20000)` 实测约 5ms，所以 1ms ≈ 4000，10ms ≈ 40000，1% ≈ 400。呼吸太快就把它调大，太慢调小。
- **一个完整呼吸是 2 秒**：0 到 100 走 100 档、每档 10ms，就是 1 秒，再 1 秒降回去。
- **`LED1` 每轮都写一次灭**，因为上一轮它可能还亮着。
- **PWM 频率 100Hz**，眼睛看不到闪；要是你觉得有轻微抖动，把 10 改成 5，`400` 改成 `200`。

模式4 就是把 `LED0` 和 `LED1` 对调，`duty` 那几个变量名沿用即可。

## 问题 6：烧录后一个常亮、一个变暗就不动了

**我：** 你这代码有问题，烧录后一个常亮，一个亮变暗就没反应了

**AI：** 两个可能，先看第一个，大概率是这个：

**你的 `soft_delay` 被编译器优化掉了。** 它是个空循环，里面没有任何副作用，Keil 开的是 -O3（`ProjectManager.CompilerOptimize=6`），这种循环很容易被整段删掉。删掉之后，PWM 里“亮”的那段时间变成 0，你就看到“亮一下 → 变暗 → 再也不动”。

### 先把 `soft_delay` 改成不会被优化掉的

```c
void soft_delay(uint32_t t)
{
	volatile uint32_t n = t;
	while (n) { n--; }
}
```

`volatile` 让编译器不敢删这个循环。加了之后速度会变，所以下面的 `400` 要重新试。

### `MODE_3` 也加固一下

```c
void MODE_3(void)
{
	static int16_t  duty = 0;      /* 改成有符号，永远不会回绕 */
	static int16_t  dir  = 1;
	static uint32_t step = 0;

	HAL_GPIO_WritePin(GPIOF, LED1_Pin, GPIO_PIN_RESET);

	if (HAL_GetTick() - step >= 10)
	{
		step += 10;
		duty = (int16_t)(duty + dir);
		if (duty >= 100) { duty = 100; dir = -1; }
		if (duty <= 0)   { duty = 0;   dir =  1; }
	}

	HAL_GPIO_WritePin(GPIOF, LED0_Pin, GPIO_PIN_SET);
	soft_delay((uint32_t)duty * 400);
	HAL_GPIO_WritePin(GPIOF, LED0_Pin, GPIO_PIN_RESET);
	soft_delay((uint32_t)(100 - duty) * 400);
}
```

两处改动：`duty` 用 `int16_t`，`<= 0` 时钳死，`100 - duty` 就不可能算出负数（负数转成 `uint32_t` 会变成 40 多亿，`soft_delay` 直接卡几分钟，灯就“没反应”了——这是第二个可能，也是常见死法）。

---

# 四、动作函数与非阻塞调度

## 问题 1：able2 按下去 LED1 亮、蜂鸣器不响

**我：** able2为啥不对？亮的是led1，蜂鸣器也不响，硬件应该没有问题

**AI：** LED1 亮和蜂鸣器不响是**两个独立问题**，因为 `able_2` 的代码里根本没写 `LED1_Pin`。

### 1. LED1 为什么会亮

那是模式1 在上一轮留下的状态（`phase==1` 时 LED1 亮）。你按 KEY1 后，主循环就停在 `able_2` 的 `HAL_Delay(2000)` 里了，`MODE_1` 整整 2 秒没有刷新，LED1 就**定格在亮**。

验证方法：按 KEY1 时，两个灯应该交替闪烁 2 秒后恢复。如果闪烁没停、照常交替，说明 `able_2` 根本没被调用；这时不用再检查蜂鸣器，问题出在调用链上。

想让它灭，加一行就行：

```c
void able_2(void)
{
	HAL_GPIO_WritePin(GPIOF, LED1_Pin,   GPIO_PIN_RESET);   /* ← 加这行 */
	HAL_GPIO_WritePin(GPIOF, LED0_Pin,   GPIO_PIN_SET);
	HAL_GPIO_WritePin(GPIOF, Buzzer_Pin, GPIO_PIN_SET);

	HAL_Delay(2000);

	HAL_GPIO_WritePin(GPIOF, Buzzer_Pin, GPIO_PIN_RESET);
	HAL_GPIO_WritePin(GPIOF, LED0_Pin,   GPIO_PIN_RESET);
}
```

### 2. 蜂鸣器不响：先确认是有源还是无源

这比极性更常见。**有源蜂鸣器**给高电平就响；**无源蜂鸣器**给直流电平永远不响，必须给方波。你板上装的是哪种，看丝印或听声音都不一定准，直接测：

临时把 `while(1)` 换成这段：

```c
while (1)
{
	/* 阶段一：常高电平 1 秒，有源蜂鸣器会响 */
	HAL_GPIO_WritePin(GPIOF, Buzzer_Pin, GPIO_PIN_SET);
	HAL_Delay(1000);

	/* 阶段二：500Hz 方波 1 秒，无源蜂鸣器会响 */
	for (uint16_t i = 0; i < 500; i++)
	{
		HAL_GPIO_TogglePin(GPIOF, Buzzer_Pin);
		HAL_Delay(1);
	}
	HAL_GPIO_WritePin(GPIOF, Buzzer_Pin, GPIO_PIN_RESET);
	HAL_Delay(300);
}
```

- 阶段一响，阶段二不响 → 有源，极性是“高响”，我的原代码没错，问题在别处
- 阶段一不响，阶段二响 → 无源，以后所有“鸣叫”都得用方波（`able_2`、`able_3` 的长鸣、基本4 都算）
- 两个都不响 → 极性反了，把 `Buzzer_Pin` 这一路的 `SET`、`RESET` 对调再试

## 问题 2：低电平有效这件事，为什么从灯的表现上看不出来

**我：** 你直接去读我目前的文件，看看为什么

**AI：** 读完了，原因很明确：**三个脚全部低电平有效**（0 = 亮或响，1 = 灭或不响）。证据来自你工程里已有的代码：

1. `gpio.c:54` 上电时把 `Buzzer_Pin|LED0_Pin|LED1_Pin` 一次性写成 `GPIO_PIN_RESET`（0）。
2. `able_2` 从头到尾**没有动过 `LED1_Pin`**，所以 LED1 一直停在 0；可你看到它**亮着**，所以 0 表示亮。
3. 同一个 `able_2` 把 LED0 拉成 `SET`（1），你看到它**灭**，所以 1 表示灭。
4. 蜂鸣器也被拉成 1，**不响**，所以 1 表示不响。

三条互相印证，没有例外。

### 为什么 MODE_1~4 看着是对的

因为极性是**整体反的**：`MODE_1` 里 `phase==0` 时 LED0 写 SET，本意是亮，实际是灭；LED1 写 RESET，本意是灭，实际是亮。交替闪看起来完全正常；`MODE_3` 的呼吸也是反的占空比，扫过去还是呼吸。

**但 `able_2` 这种“某个灯必须常亮、蜂鸣器必须响”的功能，一旦极性反了就立刻暴露。** 以后基本4（立刻关灯）也会出现同样的问题：那要求变成开灯。

### 修正后的 able_2

```c
void able_2(void)
{
	HAL_GPIO_WritePin(GPIOF, LED0_Pin,   GPIO_PIN_RESET);   /* 低 = 亮 */
	HAL_GPIO_WritePin(GPIOF, Buzzer_Pin, GPIO_PIN_RESET);   /* 低 = 响 */

	HAL_Delay(2000);

	HAL_GPIO_WritePin(GPIOF, Buzzer_Pin, GPIO_PIN_SET);
	HAL_GPIO_WritePin(GPIOF, LED0_Pin,   GPIO_PIN_SET);
}
```

### able_3 收尾那两行也有同样问题

```c
	HAL_GPIO_WritePin(GPIOF, LED1_Pin,   GPIO_PIN_SET);    /* 灭，原代码写的是 RESET=亮 */
	HAL_GPIO_WritePin(GPIOF, Buzzer_Pin, GPIO_PIN_RESET);  /* 长鸣，原代码写的是 SET=不响 */
```

`HAL_GPIO_TogglePin` 那行不用改，翻转和极性无关。

### 改完你会看到的现象

`while` 里只有 `able_2`，2 秒一到立刻进下一轮，所以你看到的是：**LED0 一直亮着、蜂鸣器一直响**，中间灭的那一下只有几微秒，看不到。这不是错，是还没接按键分发；等 `k1_flag` 那条路接上才能看到“按一下响 2 秒然后停”。

### 建议顺手加一组宏

`Mycode.h` 里：

```c
#define LED0_ON()   HAL_GPIO_WritePin(GPIOF, LED0_Pin,   GPIO_PIN_RESET)
#define LED0_OFF()  HAL_GPIO_WritePin(GPIOF, LED0_Pin,   GPIO_PIN_SET)
#define LED1_ON()   HAL_GPIO_WritePin(GPIOF, LED1_Pin,   GPIO_PIN_RESET)
#define LED1_OFF()  HAL_GPIO_WritePin(GPIOF, LED1_Pin,   GPIO_PIN_SET)
#define BEEP_ON()   HAL_GPIO_WritePin(GPIOF, Buzzer_Pin, GPIO_PIN_RESET)
#define BEEP_OFF()  HAL_GPIO_WritePin(GPIOF, Buzzer_Pin, GPIO_PIN_SET)
```

后面写基本4（关灯、关蜂鸣）时直接用 `LED0_OFF(); LED1_OFF(); BEEP_OFF();`，就不会再踩这个坑。`MODE_1`~`MODE_4` 现在不改也能跑，但里面的 SET/RESET 语义是反的，你自己心里要清楚。

## 问题 3：先放下蜂鸣器，把 able_4（基本4）写出来

**我：** 还是不行，可能是坏了，先别管这个了吧

**AI：** 好，蜂鸣器先不再处理。建议给它留一组宏，后面换模块或换引脚时只改这两行：

```c
#define BEEP_ON()   HAL_GPIO_WritePin(GPIOF, Buzzer_Pin, GPIO_PIN_RESET)
#define BEEP_OFF()  HAL_GPIO_WritePin(GPIOF, Buzzer_Pin, GPIO_PIN_SET)
```

另外，`able_3` 的收尾两行目前写反了：低电平有效时，`LED1_Pin RESET` 是点亮，`Buzzer_Pin SET` 是静音。正确写法是：

```c
	HAL_GPIO_WritePin(GPIOF, LED1_Pin,   GPIO_PIN_SET);    /* 灭 */
	HAL_GPIO_WritePin(GPIOF, Buzzer_Pin, GPIO_PIN_RESET);  /* 长鸣 */
```

### 下一步：able_4（基本4，8 分）

它的代码很短，还会顺带把“关灯”的写法固定下来：

```c
void able_4(void)
{
	HAL_GPIO_WritePin(GPIOF, LED0_Pin,   GPIO_PIN_SET);   /* 灭 */
	HAL_GPIO_WritePin(GPIOF, LED1_Pin,   GPIO_PIN_SET);   /* 灭 */
	HAL_GPIO_WritePin(GPIOF, Buzzer_Pin, GPIO_PIN_SET);   /* 停 */

	mode  = 1;
	tick  = HAL_GetTick();     /* 模式1 从"刚进入"重新开始计 500ms */
	phase = 0;
}
```

`tick`、`phase` 是你 `Mycode.c` 顶部的 static 变量，`able_4` 写在同一个文件里就能直接改；注意不要写成 `tick = 0`，否则第一次判断 `HAL_GetTick() - tick >= 500` 会立刻成立，相位会马上翻转一次。

`able_4` 是唯一**可以放在中断里直接调用**的 able：它没有延时，只有几条 `WritePin` 和赋值，几微秒就能完成。WK_UP 是报警键，按题目必须用 EXTI，正好这样处理：

```c
if (GPIO_Pin == WK_UP_Pin)
{
	able_4();      /* 重复按也没事，结果一样，不用消抖 */
}
```

它重复触发也无害，因为每次执行都是相同的最终状态，所以这里不需要 150ms 的时间窗。

下一步是把 `mode` 1~4 的分发接起来（`while` 里的 switch）。要做就说一声。

## 问题 4：able_4 里那几个变量是什么，为什么不在里面调 MODE_1

**我：** able4里面的参数是啥意思？我都没定义过，还有able4你为什么不调用mode1函数

**AI：**

### 那三个不是参数，是你自己已经定义过的变量

`able_4(void)` 没有参数。`tick`、`phase`、`mode` 是你 `Mycode.c` 里早就写好的变量：

- `Mycode.c:3` → `volatile uint8_t mode = 1;`
- `Mycode.c:10` → `static uint32_t tick = 0;`
- `Mycode.c:11` → `static uint8_t phase = 0;`

它们是**文件作用域**的变量，不在函数里面；因此同一个文件里的任何函数都能直接使用，包括 `able_4`。`able_4` 正好写在 `Mycode.c` 里，所以能直接改这三个变量。如果把它放到 `main.c` 里，就会编译不过，因为 `static` 变量别的文件看不到。

三个变量各自的作用：

| 变量 | 含义 | 谁在用 |
|---|---|---|
| `mode` | 当前是模式几，主循环 `switch` 靠它决定调用哪个 `MODE_x` | 主循环、`able_4` |
| `tick` | **上一次翻转相位的时刻**（毫秒） | `MODE_1`、`MODE_2` 用它算“过了 500ms 没有” |
| `phase` | 0 或 1，当前处在半个周期的哪一半 | `MODE_1`、`MODE_2` 用它决定谁亮谁灭 |

所以 `able_4` 里那三行，意思是：

```c
mode  = 1;                 /* 回模式1 */
tick  = HAL_GetTick();     /* 把"上次翻转的时刻"按到当前，500ms 从这一刻重新数 */
phase = 0;                 /* 回到相位0，也就是模式1刚进去时的样子 */
```

### 为什么不在 able_4 里调 MODE_1

因为这两个函数的分工不同：

- **`MODE_1` 是“画一帧”**。它只判断一次“时间到了没有”，然后按当前 `phase` 写一次灯，执行完就返回；它本身不会持续闪，靠主循环**每轮都调它**才形成闪烁。
- **`able_4` 是“改状态”**。它把 `mode`、`tick`、`phase` 改好，然后返回。

所以你按 WK_UP 之后的真实流程是：

```
WK_UP 中断 → able_4() 改好三个变量 → 中断返回
        ↓
主循环下一轮 switch(mode) → 发现 mode==1 → 调 MODE_1()
        ↓
MODE_1 每轮跑一次，500ms 翻一次 phase → 灯开始闪
```

如果在 `able_4` 里再调一次 `MODE_1()`，那只是**多画了一帧**，后面的闪烁照样要靠主循环；而且中断里做绘制，会让“谁管灯”这件事变模糊，以后改逻辑容易乱。

顺带说，也正是因为 `able_4` 只改状态、不画灯、不延时，它才可以安全地放在中断里直接调用。

## 问题 5：able_4 改了 mode 却进不去模式1，是因为还没写 switch

**我：** 我现在直接使用able4无法开启mode1是因为还没写switch对吧

**AI：** 对，你现在的状态是 `able_4` 把 `mode` 改成了 1，却没有任何代码去读取 `mode`。`main.c:102` 那行现在仍然是 `able_2();`，主循环只会执行这一句，所以 `mode` 变了也不会有人调用 `MODE_1`。

把 `while` 换成下面这样，就能让 `mode` 起作用：

```c
while (1)
{
	switch (mode)
	{
	case 1: MODE_1(); break;
	case 2: MODE_2(); break;
	case 3: MODE_3(); break;
	case 4: MODE_4(); break;
	}
}
```

`able_2` 和 `able_3` 先不要放进去，否则它们会阻塞，导致模式永远无法切换，等后面接按键分发时再加。

改完后的现象：

- 按 KEY0 时，LED0 和 LED1 在“交替闪”和“同步闪”之间切换。
- 按 WK_UP 时，立刻回到模式1，所以只在模式2/3/4 下按才看得出效果。
- 模式3/4 是呼吸，它们调到一半切走时，`duty` 会停在中途；切回来时，它会从那里继续变化，不会跳变。

写完先试按下 KEY0 和 WK_UP，有现象后再做 KEY1 和 KEY2 的分发。

## 问题 6：KEY1 和 WK_UP 怎么接，为什么一个只能置标志、一个能直接调 able_4

**我：** 现在几个灯的状态都没问题了，下一步就写按键，先做按键key1和wk-up，直接写void KEY1，检测按下后调用able2,4

**AI：** 两个键的处理方式**不一样**，原因在于 `able_2` 和 `able_4` 的性质不同：`able_2` 会阻塞 `HAL_Delay(2000)`，`able_4` 不会。

### KEY1：中断里只置标志

```c
volatile uint8_t k1_flag = 0;

void KEY1(void)
{
	static uint32_t last = 0;
	uint32_t now = HAL_GetTick();

	if (now - last > 150)
	{
		last = now;
		k1_flag = 1;
	}
}
```

**`able_2` 绝对不能在这一类函数里调用**。`able_2` 会阻塞 `HAL_Delay(2000)`；EXTI 的优先级是 0，SysTick 是 15，在中断里执行 `HAL_Delay` 时，SysTick 永远无法抢占，因此会一直等待，导致整个板子卡死。所以中断只负责设置标志，`able_2` 由主循环调用。

### WK_UP：可以直接调用

```c
void KEY_UP(void)
{
	able_4();
}
```

`able_4` 只有几条 `WritePin` 和赋值，几微秒完成，在中断里执行是安全的。抖动导致重复进入也无关紧要，因为每次结果都一样。

### 中断回调里分发

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
	if (GPIO_Pin == Key0_Pin)  { /* 你原来的 KEY0 逻辑 */ }
	if (GPIO_Pin == Key1_Pin)  { KEY1(); }
	if (GPIO_Pin == WK_UP_Pin) { KEY_UP(); }
}
```

你原来的 `Key0` 那段直接复制到第一个 `if` 里就行。

### 主循环加两句

```c
while (1)
{
	if (k1_flag)
	{
		k1_flag = 0;
		able_2();
		k1_flag = 0;        /* 这 2 秒里按的都丢弃，实现"鸣叫中再按忽略" */
	}

	switch (mode)
	{
	case 1: MODE_1(); break;
	case 2: MODE_2(); break;
	case 3: MODE_3(); break;
	case 4: MODE_4(); break;
	}
}
```

注意 `k1_flag` 要加 `volatile`，并在 `Mycode.h` 里加一行 `extern volatile uint8_t k1_flag;`，否则 `main.c` 看不到它。

### 一个附带的好处

`KEY_UP` 在中断里执行，所以**即使 `able_2` 正在响那 2 秒，按 WK_UP 也能立刻关灯**（`able_2` 卡在 `HAL_Delay` 里也拦不住中断）。这正好是基本4 要求的“立刻”。

写完烧进去，按 KEY1 应该看到 LED0 亮 2 秒然后恢复，按 WK_UP 立刻回模式1。

## 问题 7：变量定义在 Mycode.c，main.c 读不到

**我：** 变量定义呢，主函数读不到变量

**AI：** `main.c` 里的 `#include "Mycode.h"` 已经有了（`main.c:25`），缺的是**在头文件里声明**。分两处改：

### `Mycode.c` 顶部加定义（已有 `mode` 那一带）

```c
volatile uint8_t mode = 1;
volatile uint8_t k1_flag = 0;      /* ← 加这行 */
```

### `Mycode.h` 里加声明

```c
extern volatile uint8_t mode;
extern volatile uint8_t k1_flag;   /* ← 加这行 */

void KEY1(void);
void KEY_UP(void);
```

没有 `extern` 那行的话，`main.c` 编译时就会报 `k1_flag` 未定义，因为 `main.c` 看不到 `Mycode.c` 里的变量。`extern` 只是告诉编译器“这个变量在别的文件中定义”，链接时会自动对上。

要点：

- 定义只写一次，在 `Mycode.c` 里写 `volatile uint8_t k1_flag = 0;`
- 其他文件用之前，在 `Mycode.h` 里写 `extern volatile uint8_t k1_flag;`
- 函数也一样，定义在 `.c`，原型放 `.h`
- `Mycode.c` 自己也要 `#include "Mycode.h"`（已经有，`Mycode.c:1`），所以定义写在 `#include` 之后没问题

改完编译一下，`k1_flag` 和 `KEY1()` 在 `main.c` 里就能用了。

## 问题 8：KEY1 不灵敏、WK_UP 看不出效果

**我：** 现在上电默认mode1,按了key1有反应，但是不灵敏，有时候按了就没反应。wkup现在看不出来，因为一直是mode1,现在写key0代码

**AI：**

### KEY0

```c
void KEY0(void)
{
	static uint32_t last = 0;
	uint32_t now = HAL_GetTick();

	if (now - last > 150)
	{
		last = now;
		mode = (mode == 1) ? 2 : 1;
	}
}
```

回调里改成：

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
	if (GPIO_Pin == Key0_Pin)  { KEY0(); }
	if (GPIO_Pin == Key1_Pin)  { KEY1(); }
	if (GPIO_Pin == WK_UP_Pin) { KEY_UP(); }
}
```

`Mycode.h` 里补一行 `void KEY0(void);`。

### KEY1 不灵敏：两个原因，都对得上

**原因一：`150` 太宽了。** 这个数是给机械抖动留的余量，但普通按键的抖动只有几毫秒。如果把它设成 150ms，你两次按下只要间隔小于 0.15 秒，第二次就会被那个窗口挡掉。改成 `30`：

```c
	if (now - last > 30)
```

这三个键都改成 30，消抖够用，又不会挡掉正常的连续操作。

**原因二：`able_2` 那 2 秒里按下的按键都会被丢弃。** 这不是 bug，是题目“鸣叫中再按忽略”的要求：

```c
	k1_flag = 0;
	able_2();
	k1_flag = 0;      /* ← 这一句把 2 秒内的按下全清了 */
```

所以如果刚按完 KEY1，在响的那 2 秒里再按，肯定没有反应。等 2 秒结束后按才算数。如果你觉得“按了没反应”，先确认是不是正好在这 2 秒内。

### WK_UP 的验证方法

现在一直停在 mode1，所以按 WK_UP 看不出变化是正常的，因为它只是把 `mode` 又设成 1。要验证它，就先按 KEY0 切到模式2（同步闪），再按 WK_UP，灯应该立刻变回交替闪。这样就能看出效果了。

## 问题 9：按键十次丢七次，把 able_2 改成 start / busy / tick 三段式

**我：** key1也装上，情况还是差不多，大概10次按键有7次都会丢失

**AI：** 先别怀疑按键。你现在的主循环是**这样卡住的**：

```c
if (k1_flag) { k1_flag = 0; able_2(); k1_flag = 0; }
```

`able_2` 里有 `HAL_Delay(2000)`，这 2 秒里整个主循环**完全停住**，`MODE_x` 不跑，`k1_flag` 不检查。所以：

- 这 2 秒里按 KEY1、KEY0、WK_UP 之外的任何键，全部无效；
- 2 秒结束时 `k1_flag = 0` 把期间按下的全清掉；
- 你如果大概每 0.6 秒按一次，就会有 2/3 左右的按键落在窗口里，这就是“10 次丢 7 次”。

### 先确认一下（10 秒）

按一次 KEY1 → 等 LED0 那 2 秒亮完 → 停 3 秒 → 再按。这样按 10 次：

- **10 次都灵** → 按键和中断都没问题，就是这个阻塞窗口吞的；
- **还是丢** → 那才是中断本身的问题，回头查 `PC2` 的接线和边沿。

### 根治：把 able_2 改成非阻塞

这一改，2 秒里主循环照转，所有按键都灵，而且拓展3 的“多击和呼吸同时成立”也要靠它。

`Mycode.c` 里：

```c
static uint8_t  act2    = 0;
static uint32_t act2_t0 = 0;

void able_2_start(void)          /* 启动，只跑一瞬 */
{
	act2    = 1;
	act2_t0 = HAL_GetTick();

	HAL_GPIO_WritePin(GPIOF, LED0_Pin,   GPIO_PIN_RESET);
	HAL_GPIO_WritePin(GPIOF, Buzzer_Pin, GPIO_PIN_RESET);
}

uint8_t able_2_busy(void)        /* 是否还在响 */
{
	return act2;
}

void able_2_tick(void)           /* 每轮调，只管"该停了没有" */
{
	if (act2 == 0) return;

	if (HAL_GetTick() - act2_t0 >= 2000)
	{
		act2 = 0;
		HAL_GPIO_WritePin(GPIOF, Buzzer_Pin, GPIO_PIN_SET);
	}
}
```

主循环改成：

```c
while (1)
{
	if (able_2_busy())
	{
		k1_flag = 0;             /* 响的时候按的，忽略 */
		able_2_tick();
	}
	else
	{
		if (k1_flag) { k1_flag = 0; able_2_start(); }

		switch (mode)
		{
		case 1: MODE_1(); break;
		case 2: MODE_2(); break;
		case 3: MODE_3(); break;
		case 4: MODE_4(); break;
		}
	}
}
```

关键点：响的时候**不进 `switch`**，所以 `MODE_x` 不会把 LED0 抢回去；2 秒一到 `act2` 清零，主循环自然回到 switch，LED0 交还给模式。

`able_4` 里再加一句 `act2 = 0;`，这样 WK_UP 在响的过程中也能立刻停掉。

头文件里把 `able_2` 的原型换成 `able_2_start` / `able_2_tick` / `able_2_busy` 三个。

改完再试：这 2 秒里按 KEY0 应该能立刻切模式，按 KEY1 依然被忽略（题目要求），这才对。

## 问题 10：able2 变成“维持当前状态 2 秒”，按键照旧丢

**我：** 按键还是丢失（我没有在两秒内点击），10丢10都有发生，纯看运气。而且，还有新问题，able2的现象变成了“维持当前状态2s”，意思就是说：如果我按下瞬间led0.1都亮/灭着，那就都亮/灭2s，不符合able2的led0亮，1灭

**AI：** 第二个现象暴露了真正的 bug。看主循环这一轮发生了什么：

```
else 分支
  ├─ if (k1_flag) { k1_flag = 0; able_2_start(); }   ← 点亮 LED0、灭 LED1
  └─ switch (mode) { case 1: MODE_1(); }             ← 立刻又把两个灯写回去了！
```

`able_2_start()` 刚把两个灯设好，**同一轮里紧接着执行的 `switch` 就让 `MODE_1` 把 LED0 和 LED1 覆盖了**。所以 `able_2_start` 对灯的画面只作用了几微秒。下一轮 `able_2_busy()` 为真，主循环不再调用 `MODE_x`，于是画面停在 `MODE_1` 最后写入的状态，这就是你看到的“维持当前状态 2s”。

### 改法：加一个 `else`

```c
	else
	{
		if (k1_flag) { k1_flag = 0; able_2_start(); }
		else
		{
			switch (mode)
			{
			case 1: MODE_1(); break;
			case 2: MODE_2(); break;
			case 3: MODE_3(); break;
			case 4: MODE_4(); break;
			}
		}
	}
```

本轮启动后**不进入 switch**，下一轮会被 `able_2_busy()` 接走，灯就保持 `able_2_start` 设的样子；2 秒后 `act2` 清零，主循环自动回到 switch。

### 这个问题也能解释大部分“按键丢失”

因为 `able_2` 现在实际上什么都不改，只是把画面冻结 2 秒。如果你的模式正好处于“两个灯都灭”的那半个周期（模式2 的灭相），冻结和平常一模一样，看起来就像按了没反应。

所以顺序是：**先修这个 else，再测按键**。修完之后，每次按下都应该看见 LED0 立刻亮、LED1 立刻灭，响 2 秒再恢复。这样“有没有响应”就一目了然了。

### 如果修完还有真正的丢键

那才是中断本身的问题，用一个 30 秒的测试直接判定。临时把回调改成：

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
	if (GPIO_Pin == Key1_Pin) { HAL_GPIO_TogglePin(GPIOF, LED0_Pin); }
}
```

主循环改成：

```c
while (1)
{
}
```

空循环，没有任何东西去覆盖 LED0。这时**每按一次 KEY1，LED0 就翻转一次**。按 10 次：

- 10 次都翻 → 中断 100% 可靠，问题全在主循环逻辑里
- 少翻几次 → 是硬件边沿问题（PC2 接线、按键接触、内部下拉太弱），得从电路上找

先改 else，把结果告诉我。

## 问题 11：下载弹 could not load file .axf，其实是 iif 拼写错误

**我：** ？你动我文件了？

**AI：**

### 先回答：我没动你的文件

我这次只调用了两个只读操作：`read` 和 `Get-ChildItem`（列目录）。这个环境里，我手上的文件写入权限也只覆盖我自己那个工作目录（`D:\DSH`），你的工程对我来说是只读的。

时间戳也能对上：

- `Mycode.c` 最后修改 **21:42:43**
- `main.c` 最后修改 **21:50:50**（这是你自己保存的时间）
- `MDK-ARM\project\project.hex` 生成于 **21:42:44**

这些都是你自己在 Keil 里保存和编译的时间，我读文件不会改这个时间。

### 这个报错是编译失败，不是硬件问题

`project.axf` 现在**根本没生成**：`MDK-ARM\project\` 里只有 21:42 的 `.hex` 和 `.map`，没有 `.axf`。Keil 执行下载时找不到文件，就弹出这个框。

### 错误在 `main.c:109`

```c
iif (k1_flag) { k1_flag = 0; able_2_start(); }
```

多打了一个 `i`，把 `if` 写成了 `iif`。这才是编译不通过的原因。

改掉之后：

1. 按 **F7** 编译，先确认下面 Build Output 是 `0 Error(s)`；
2. 再按 **F8** 下载。

以后养成习惯：先看 Build Output 有没有红字，再点下载。像刚才这样“改完直接下载”，报的就是这种误导性的错误。

### 顺手一个小清理

`Mycode.h:22` 和 `Mycode.h:23` 重复了两行 `extern volatile uint8_t mode;`。这不会报错，但删掉一行更干净。

改完编译、下载，再看 able_2 是不是“LED0 亮、LED1 灭、持续 2 秒”。

## 问题 12：还剩随机丢键，改用状态锁存

**我：** 现在好多了，按键丢失还是有，但概率低多了，10丢3，10丢0，10丢6这样子，able2也没问题了。反正现在总体上可以了。

**AI：** 先把两种“丢”分开：

- **在响的那 2 秒里按的** → 那 2 秒里按下的按键本来就会被忽略（题目要求“鸣叫中再按忽略”），程序有意丢弃它，这不算丢键。
- **不在响的那 2 秒里按，仍然没反应的** → 才是真丢。

你如果连续测试，每次成功按下的按键都会让板子进入那 2 秒的忽略窗口；如果你大概每秒按一次，10 次里有 5 到 6 次落在窗口里，这是正常的。按 10 次丢 0 次，说明你中间等过。

### 如果确认在不响的那 2 秒里按也丢，就换成状态锁存

比 30ms 时间窗更稳，不会误挡快速连按：

```c
volatile uint8_t k1_flag   = 0;
static uint8_t   k1_locked = 0;

void KEY_1(void)          /* 中断里，只锁存 */
{
	if (k1_locked == 0)
	{
		k1_locked = 1;
		k1_flag   = 1;
	}
}

void KEY_1_release(void)  /* 主循环每轮调 */
{
	if (k1_locked && HAL_GPIO_ReadPin(GPIOC, Key1_Pin) == GPIO_PIN_RESET)
		k1_locked = 0;
}
```

主循环最上面加一句 `KEY_1_release();`（放在 `if (able_2_busy())` 前面，响的时候也要执行）。

好处是：**按下沿立刻生效，不用等 30ms**；抖动产生的多次进入被 `k1_locked` 挡住；只有真正松开后才允许下一次。`KEY_0` 也可以照这个改。

### 硬件上再加一层（可选）

按键两端并一个 **0.1µF 电容**，这是电赛现场最常用的硬件消抖，能把随机丢键压到几乎为零。面包板加长杜邦线最容易出接触问题，先看看是不是某根线的接触问题。

---

# 五、拓展二：模式5 与多击检测

## 问题 1：五个按键已经分完了，模式 5 从哪里进

**我：** 我现在手里有5个按键，题目说了key0控制mode12，就别动了，免得说我不按要求来，key12也有对应功能，wkup也是。最后一个key3我想留给题目没说的如何进入mode3,4，也就是说key3功能类似key1

**AI：** 好，PF8 已经在引脚配置里了，`EXTI9_5_IRQHandler` 里也已经带 `Key3_Pin`。这两个地方我核对过，CubeMX 不用动，你只需要加一个函数和一行分发。

```c
void KEY_3(void)
{
	static uint32_t t3 = 0;

	if (HAL_GetTick() - t3 > 30)
	{
		t3 = HAL_GetTick();

		if      (mode == 3) mode = 4;
		else if (mode == 4) mode = 5;
		else                mode = 3;
	}
}
```

**这里我建议带上模式 5**：拓展 2 写的是"在模式 5 下 KEY2 改多击"，模式 5 必须有入口；5 个按键已经分完（KEY0 切 1、2，KEY1→able_2，KEY2→able_3，WK_UP→able_4），只剩 KEY3，让它顺带管到 5 是唯一不违反"KEY0 只切 1、2"的做法。

## 问题 2：给一份 mode5 和 KEY2 相关的代码

**我：** 在给一下mode5和key2相关代码

**AI：**

### 1. 变量与中断里的 KEY_2

```c
volatile uint8_t k2_cnt    = 0;     /* 这一轮点了几下 */
static   uint8_t k2_locked = 0;     /* 这一下已记过，等松开 */
static   uint32_t k2_last  = 0;     /* 最后一次按下的时刻 */

void KEY_2(void)
{
	if (k2_locked == 0)
	{
		k2_locked = 1;
		k2_cnt++;
		k2_last = HAL_GetTick();
	}
}
```

### 2. 主循环里的结算

```c
#define K2_GAP  400                 /* 两击之间超过 400ms 就结算 */

void key2_tick(void)
{
	/* 松开才解锁 */
	if (k2_locked && HAL_GPIO_ReadPin(GPIOC, Key2_Pin) == GPIO_PIN_RESET)
		k2_locked = 0;

	/* 鸣叫中按的，扔掉 */
	if (able_2_busy())
	{
		k2_cnt = 0;
		return;
	}

	/* 安静够久了，结算 */
	if (k2_cnt > 0 && (HAL_GetTick() - k2_last) > K2_GAP)
	{
		uint8_t n = k2_cnt;
		k2_cnt = 0;

		if      (n == 1) able_2_start();   /* 单击 → 基本2 */
		else if (n == 2) able_3();         /* 双击 → 基本3 */
		else             able_4();         /* 三击及以上 → 基本4 */
	}
}
```

`Key2_Pin` 在 **GPIOF** 上，但 `HAL_GPIO_ReadPin` 的第一个参数是端口。这一句写 `GPIOF`，不是 `GPIOC`：

```c
if (k2_locked && HAL_GPIO_ReadPin(GPIOF, Key2_Pin) == GPIO_PIN_RESET)
```

### 3. 模式 5

```c
void MODE_5(void)
{
	static int16_t  duty = 0;
	static int8_t   dir  = 1;
	static uint32_t step = 0;

	if (HAL_GetTick() - step >= 10)
	{
		step += 10;
		duty += dir;
		if (duty >= 100) { duty = 100; dir = -1; }
		if (duty <= 0)   { duty = 0;   dir =  1; }
	}

	HAL_GPIO_WritePin(GPIOF, LED0_Pin, GPIO_PIN_RESET);   /* 亮 */
	soft_delay(duty * 400);
	HAL_GPIO_WritePin(GPIOF, LED0_Pin, GPIO_PIN_SET);     /* 灭 */
	soft_delay((100 - duty) * 400);

	HAL_GPIO_WritePin(GPIOF, LED1_Pin, GPIO_PIN_SET);     /* LED1 灭 */
}
```

`RESET` 亮、`SET` 灭，跟你 able_2 的低电平点亮一致。

### 4. 测试顺序

进模式 5（KEY3 按两下：1→3→4→5）：点 1 下 → LED0 常亮 + 蜂鸣 2 秒；快点 2 下 → LED1 闪 3 秒；快点 3 下 → 全灭、回模式 1。点得不够快就会拆成两次单击。

## 问题 3：KEY2 无论怎么按都只算单击

**我：** 说一下目前状态，key2没有多击能力，无论怎样都是单击（able2），另外有一个发现，其他四个按键还是有丢失(我说的丢失都是2s外的)现象，但是key3完全没有丢失现象，无论快慢点击都能立马响应，这是为什么

**AI：** 两个问题分开说。

### 一、KEY2 永远变成单击

问题出在 `K2_GAP = 400` 太短：结算发生在**最后一次按下之后 400ms**。如果你两次点击的间隔超过 400ms（人对这种硬按键"双击"的间隔常常是 400~600ms），第一次点击就会先结算成单击，第二次点击时 `able_2` 已经开始了，2 秒内的点击全被 `k2_cnt = 0` 清掉。所以看起来永远是单击。先把它改成 **600**：

```c
#define K2_GAP  600
```

测试时请按这个节奏：**三下在 1.2 秒内点完**。如果还是不行，就加一个调试用的变量，在 Keil 调试里看 `k2_cnt` 到底数到了几：

```c
volatile uint8_t k2_dbg = 0;        /* 结算时记下 n，调试用 */

/* key2_tick 里 */
uint8_t n = k2_cnt;
k2_cnt = 0;
k2_dbg = n;                          /* 加这一行 */
```

如果 `k2_cnt` 数到了 3 但走的是单击分支，那是分发逻辑的问题；如果只数到 1，那就是两次点击之间 `k2_locked` 没有解开。

### 二、KEY3 不丢的原因

对比一下这四个按键的差别：

| 按键 | 什么时候真正动作 | 有没有额外门槛 |
|---|---|---|
| KEY3 | **中断里当场改 mode** | 只有 30ms 窗口 |
| KEY0 | 中断里当场改 mode | 有 **150ms** 窗口 |
| KEY1 | 置 flag，等主循环来取 | 30ms 窗口 + **`able_2` 的 2 秒里被 `k1_flag = 0` 吃掉** |
| KEY2 | 置计数，**等 400ms 结算** | 400ms 窗口 + 2 秒内被清掉 |

KEY3 之所以从不丢失，原因有两个：**它当场生效**，边沿进来那一刻就改了 mode，不需要主循环配合；**它的窗口最短（30ms）**，只挡抖动，挡不住人手。

## 问题 4：三击只出 able_2，是不是丢了两下

**我：** 又试了几次，发现key2能工作，但是应该还是因为按键丢失的原因，进入able3和4的概率特小，应该就是：我三击，但是丢失了俩，导致进入了able2

**AI：**

### 问题不在按键丢失，而在 able_2 提前被触发

看 `key2_tick` 的顺序：`if (k2_cnt > 0 && (HAL_GetTick() - k2_last) > K2_GAP)`（600ms），规则是：**只要距离上一次按下超过 600ms，就认为你已经点完了。**

所以你点三下时，如果第一下和第二下之间隔了 600ms 以上，程序在第一下之后 600ms 就判定"这是单击"，于是调用 `able_2_start()`。`able_2` 一开始，接下来 2 秒里 `key2_tick` 第一步就是：

```c
if (able_2_busy()) { k2_cnt = 0; return; }   /* 后面两下全被清掉 */
```

后两下被清掉，所以你看到的永远是 able_2。这不是按键丢键，而是 **able_2 提前触发，后两下被它吞掉**。这也能解释为什么"able_3/4 概率特小"：只有当你三下点得非常快、每下间隔都在 600ms 以内时，程序才会一次收到 3 次按键。

### 改两个数

```c
#define K2_GAP  700        /* 600 → 700，手慢也能凑上 */
```

顺手把两个跨中断的变量加上 `volatile`：

```c
volatile uint8_t  k2_cnt    = 0;
static volatile uint8_t  k2_locked = 0;
static volatile uint32_t k2_last   = 0;
```

`k2_last` 在中断里写、在主循环里读，不加 `volatile` 的话 -O3 可能把它优化成只读一次，那样结算会永远卡在旧值上。

### 验证一下到底是哪边

```c
volatile uint8_t k2_irq = 0;

void KEY_2(void)
{
	k2_irq++;                    /* 每次进中断就 +1 */
	if (k2_locked == 0)
	...
}
```

连点三下看 `k2_irq`：**= 3 但 n 只有 1** → 按键没丢，是结算窗口把它拆成了三次单击，接着调 `K2_GAP`；**< 3** → 真的是按键或中断丢，回到硬件那条线。按我的判断，八成是第一种。

点三下要**总共控制在一秒半以内**，700 还偏紧就放到 800。上限是：`K2_GAP` 越大，单击响应越慢（点一下要等 700ms 才响），这是"单击要等多久"和"多击能点多快"之间的取舍，答辩时可以这么说。

## 问题 5：按键丢失到底为什么会发生

**我：** 先别管功能了，现在给我解释一下，到底为什么会发生按键丢失（不是题目强制等待2s导致的那种，就是我正常按了按键没反应）？我之前只遇到过按了多次触发（没消抖导致的），但为什么这次的代码会这么频繁的发生按键丢失

**AI：**

### 一句话结论

抖动不只表现为"按一下触发好几次"，同样会表现为"按了却一次都没触发"。**你现在用的判定方式，恰好是最容易把一次按下变成零次的那一种。**

### 一、按下那一刻，引脚上其实发生了什么

你以为按下是引脚电平从 0 变成 1 的一次跳变，实际上电气上是一串跳变：轻触开关会经历 **"碰上—弹开—再碰上—再弹开……"**，第一次接触往往只有几十纳秒到几微秒。关键点是：**最早的那一两个脉冲非常窄，中间那段是"断开"**，如果程序只看那一瞬间，就等于把整次按下的判定权交给了最不可靠的瞬间。

### 二、边沿触发只承认"那一瞬间"

你配置的是 **内部下拉 + 上升沿触发**。引脚静止时被拉到 0V，只有触点闭合的那一下才会出现 `0→1`。而 STM32 的 EXTI 边沿检测是按时钟采样：PCLK2 = 16MHz，即每 62.5ns 看一眼。于是：

- 擦碰脉冲**宽于**一个时钟周期且幅度越过阈值 → 被看到了，触发一次。
- 擦碰脉冲**窄于**一个采样周期，或者电压爬到一半又掉下去没越过 2.3V 的输入门限 → **完全没被看到**。而这之后引脚又回到 0，**再也不会出现 `0→1`**，直到你下一次真的按下去。

这就是"零触发"的来源。不是因为按得不够重，而是那次按下在电气上只产生了一个窗口极窄的脉冲，而程序恰好错过了这唯一的信号。

**边沿触发记录的是"变化那一瞬"，而不是"变化之后的状态"。记录瞬间等于冒险，记录状态才是安全的。**

### 三、还有三种纯软件丢弃，让你误以为按键丢失

这几种和硬件无关，是确定性的，值得单独列出来：

1. **`> 30ms` 的时间窗**。它限制的是"两次按下之间至少隔多久"，不是消抖。两次快速连按，第二下直接丢掉。
2. **`able_2` 响的那 2 秒里**，`k1_flag = 0`、`k2_cnt = 0` 是主动清空的。你说过不算这种，但它确实在制造"按了没反应"。
3. **`k2_locked` 可能永久卡死**。它只在"读到引脚为低"时才解锁。如果某次松手时，引脚因为虚接、氧化或者你手还按着半边，没有可靠回落到低电平，这个锁就**再也解不开**，之后所有按下全部无效——表现就是"突然彻底没反应，怎么按都不行"。你描述的正是这个现象的其中一次。

第 3 条是这批代码里最容易被忽略的一点。它把"一次异常"变成了"永久失效"。

### 四、正确的做法

把判定依据从"边沿"换成"电平"：**中断只负责"有人动了"**，置一个标志就退出；**主循环里读电平**，连续几次读到高才算真按下。这样判的是"状态"而不是"瞬间"，哪怕抖动把第一个脉冲打得很窄，只要手指还按着，下一个采样就会看到高电平，事件不会丢。

### 五、KEY3 不丢失的原因

代码上没有本质区别，同样是 30ms 窗口、同样进 EXTI9_5，唯一的差别是**那个物理按键和那根线**。

## 问题 6：那要改按键的模式吗

**我：** 意思是我现在需要去改按键的模式对吧

**AI：** 对，要改。不过不在引脚配置上：引脚仍然保持中断（**EXTI 照旧，不影响"报警类必须用中断"这条评分要求**），改的是**在中断里做什么**。

1. **中断里只置一个标志**，比如 `k0_hit = 1`；`> 30` 的时间窗、`k2_locked`、读电平解锁全部从里面删掉，中断只做通知、不做判定。
2. **主循环里统一按键扫描，按"电平"判定**：每轮读一次引脚，连续两次（约 10~20ms）都读到同一个电平，才承认状态真的变了。这是标准的软件消抖，判定的是**状态**，不是那一瞬间。
3. **键的状态用锁，不用窗口**：按下时置锁，只有确认松开了才解锁。这样按住不会重复触发，两键之间也互不影响，更不会像 `k2_locked` 那样把自己卡死。

改完之后 `30ms`、`150ms`、`K2_GAP` 的那套时间窗和 `k2_locked` 都能删掉，代码总量反而比现在**少**——你现在觉得"复杂"，恰恰是因为每个键各有一套独立的消抖方式。

先跑那个**中断指示灯测试**（主循环清空，中断里 `HAL_GPIO_TogglePin(LED1)`）：按 10 下翻 10 次，说明硬件没丢、全是软件造成；只翻 7 次，说明**电气层真的丢脉冲**，软件只能减少、不能根治。

不是"把按键从中断改成轮询"，而是"**中断负责报告，电平负责判定**"。

## 问题 7：中断模式下没有电平触发

**我：** 咋改？中断模式下没有电平触发

**AI：** 你没找错地方：**STM32 的 EXTI 硬件本来就只有边沿触发，没有电平触发**，所以下拉框里不会出现这个选项。这是硬件限制，不是 CubeMX 漏了。

我说的"电平判定"不是 CubeMX 里的选项，而是**代码里用 `HAL_GPIO_ReadPin()` 读出来的**：硬件仍是上升沿触发，引脚一动就进中断；软件在中断里再去读一下当前电平，决定"真按了还是抖了一下"。CubeMX 保持 `Rising edge` + `Pull-down`，不用改。

```c
/* 中断里：只通知 */
if (GPIO_Pin == Key2_Pin) k2_hit = 1;

/* 主循环里：读电平做判定 */
if (HAL_GPIO_ReadPin(GPIOF, Key2_Pin) == GPIO_PIN_SET)
{
	/* 引脚现在确实是高电平——真的按着 */
}
```

中断负责"快"，电平负责"准"。

### 那个双边沿选项要不要选

`Rising/Falling edge trigger detection`（双边沿）看着像能防丢，其实帮不上：你丢的是"按下那一瞬间的沿"，松手时的下降沿**救不回已经丢掉的那次按下**，而且双边沿会让抖动产生双倍中断。**不要选**，保持上升沿。

## 问题 8：说详细一点，不知道改啥

**我：** 说详细一点，不知道改啥

**AI：** 好，给你一份"照着替换"的清单。

### 一、先删掉这三样东西

| 删什么 | 在哪 | 为什么 |
|---|---|---|
| `static uint8_t k2_locked = 0;` | `Mycode.c:191` | 它只读到低电平才解锁，一旦某次松手没回落到低谷，就**永久卡死**，之后所有按键全部失效 |
| `KEY_2` 里的 `if (k2_locked == 0) { k2_locked = 1; ... }` | `194-202` | 同上 |
| `key2_tick` 开头那三行解锁代码 | `207-209` | 同上 |

那个 `k2_locked` 就是你遇到的"怎么按都没反应"，它不是概率问题，而是**一次卡死就永久失效**。

### 二、为什么这么改就不丢了

**1. 从"单 bit 标志"改成"计数器"。** 原来 `k1_flag = 1` 是一个比特。如果主循环正卡在 `MODE_1` 的 `HAL_Delay(500)` 里，你在这个 500ms 里按了两下，两次都把它置成 1，**两下合并成一下**，这就是"按了两次只响应一次"。换成计数后，主循环回来时会把积压的事件一笔一笔处理掉，一个都不丢。

**2. 删掉 `k2_locked`。** 它把"这一次读错了"变成"以后永远读不到"。`KEY_3` 从不丢，正是因为它没有这种会锁死的状态。现在所有键都用同一套"30ms 窗口 + 计数"，行为和 KEY3 完全一致。

**3. 30ms 窗口只挡抖动，不挡人。** 人手最快的连击间隔也在 60ms 以上，30ms 的窗口碰不到正常操作。

## 问题 9：KEY0 和 KEY3 代码一样了，为什么表现还是不一样

**我：** 我现在就是30，之前改过，你能不能看完我的代码在说话，别乱猜行不行

**AI：** 你说得对，我刚才用的是你改之前的版本。现在读了当前文件，`Mycode.c:172` 确实是 `> 30`。

### 现在的关键问题

你的 `KEY_0` 和 `KEY_3` 现在**代码已经完全一样了**：

```c
KEY_0:  if (now - last > 30) { last = now; mode = (mode == 1) ? 2 : 1; }
KEY_3:  if (HAL_GetTick() - t3 > 30) { t3 = ...; if (mode==3) ... }
```

都是 30ms 窗口、都在中断里立刻改 `mode`、都不读电平。剩下能解释差别的地方只有三个：**PC0 和 PF8 是不同的物理引脚**；**两个键的动作不同**——KEY3 是 3→4→5→3 **单向前进**，KEY0 是 1→2→1 **来回切换**；也可能只是观察方式。

### 第 2 点很可能是"假丢失"

你按 KEY0 是在 1 和 2 之间来回切换：**连按 10 下如果都成功了，模式会回到原样**，从外面看就像"一次都没动"，而且"交替闪"和"同步闪"的差别要闪一下才看得明显。所以用"连按 10 下看变没变"测 KEY0，**本来就测不出来**。

### 换个测法

别连按，**一下一下按**，每按一次停下来看一秒。如果每次单按都在换，那 KEY0 一次没丢，之前看到的"丢"是**两次按下的效果互相抵消了**；如果按下去纹丝不动、等两秒也不变，那才是真丢——而代码这边没有任何可改的，`KEY_0` 和从不丢的 `KEY_3` 已经完全同构，差别只剩 PC0 那个物理引脚。

## 问题 10：结论——是按键本身坏了

**我：** 好吧，我试了。终于明白了，我的按键有问题（硬件），key3用的室友的，所以没有问题

**AI：** 结案，和我前面推的方向一致：`KEY_0` 的代码和 `KEY_3` 已经一样，问题在 PC0 那个物理按键上。这也解释了"按了没反应"和"连按两次才动一下"为什么能同时出现。

### 代码里还有一个与坏键相关的真实隐患

`Mycode.c:191` 的 `k2_locked`：

```c
static volatile uint8_t k2_locked = 0;
...
if (k2_locked && HAL_GPIO_ReadPin(GPIOF, Key2_Pin) == GPIO_PIN_RESET)
	k2_locked = 0;
```

如果某次松手时引脚因接触不良没回落到低电平，这个锁就**永久解不开**，之后 KEY2 怎么按都没反应。坏按键最容易触发这个状态。删完后 `KEY_2` 改成：

```c
void KEY_2(void)
{
	static uint32_t dbj = 0;
	uint32_t now = HAL_GetTick();

	if (now - dbj > 30)
	{
		dbj = now;
		k2_cnt++;
		k2_last = now;
	}
}
```

30ms 窗口本身就足以挡住抖动，不需要那个锁。

### 还有一个功能缺口

你现在 `key2_tick` 在**所有模式下**都走多击，但题目拓展 2 说的是"**模式 5 下** KEY2 才改多击"，基本功能 3 要求的是"KEY2 → 3 秒倒计时"（12 分）。也就是说，模式 1~4 下按 KEY2 现在什么都不发生，基本 3 没入口了。在 `key2_tick` 里加一个判断就够：

```c
if (mode != 5)
{
	if (k2_cnt > 0) { k2_cnt = 0; able_3(); }
	return;
}
```

## 问题 11：模式 1~4 下 KEY2 应该是单击

**我：** 你上面说的我没改，也就是我现在的代码什么都没动。我把原来的接"坏按键"的按键都用好按键替换了，目前是全都没有问题，key0123,wkup都能按预期执行（还没找到好的蜂鸣器）。然后我现在又读了一遍题目，发现目前的逻辑不符合题目。目前：key2为多击。题目：key2为单击，由key3进入mode5后改为多击

**AI：** 对，题目是「模式 1~4 下 KEY2 就是基本 3 的那个键，进了模式 5 才变成多击」。你现在的 `key2_tick` 不分模式，所以补一个分支就行。整段替换成：

```c
void key2_tick(void)
{
	/* 松开才解锁 */
	if (k2_locked && HAL_GPIO_ReadPin(GPIOF, Key2_Pin) == GPIO_PIN_RESET)
		k2_locked = 0;

	/* 模式 1~4：KEY2 就是普通单击 → 基本3 的三秒倒计时 */
	if (mode != 5)
	{
		if (k2_cnt > 0)
		{
			k2_cnt = 0;
			able_3();
		}
		return;
	}

	/* 模式 5 下面才是多击 */

	if (able_2_busy())
	{
		k2_cnt = 0;
		return;
	}

	if (k2_cnt > 0 && (HAL_GetTick() - k2_last) > K2_GAP)
	{
		uint8_t n = k2_cnt;
		k2_cnt = 0;

		if      (n == 1) able_2_start();
		else if (n == 2) able_3();
		else             able_4();
	}
}
```

`KEY_2` 那个函数不用动，`k2_locked` 也不用动。

### 这样改完的行为

| 模式 | 按 KEY2 |
|---|---|
| 1、2、3、4 | **立刻**触发 3 秒倒计时（LED1 以 1Hz 闪），不用等结算窗口 |
| 5 | 单击 → 基本2，双击 → 基本3，三击 → 基本4 |

模式 1~4 下让它**立刻**响应、不走 600ms 结算：既然是单击就没必要等，按下就响，也符合"响应要快"。

### 一个以后要处理的地方

模式 5 下双击走的 `able_3()` 是阻塞的 3 秒，这 3 秒 `MODE_5` 不跑、呼吸会停，做拓展 3 时要改成非阻塞。

---

# 六、拓展一：GPIO 输出方波与定时器约束

## 问题 1：模式 6 的引脚、频率、步进先定三个数，方波怎么不阻塞主循环地输出

**我：** 现在差不多没问题了（除了纯硬件问题），最后还有一个gpio输出方波，步进学长说设为5%，这个模式也需要按键，你看看怎么设置。先不用给代码，先讲讲怎么实现和逻辑

**AI：** 这块的难点在于你已经使用 GPIOC 的 PC0、PC2 两个按键，再加上 PF 口的全部引脚，空闲引脚所剩不多，而并非方波本身。方波只要用 GPIO 翻转就能产生，真正需要设计的是**怎么在不阻塞主循环的前提下输出**，以及**按键从哪来**。下面先按顺序讲一遍。

### 一、先定三个数

**引脚**：建议用 **PC1**。它夹在 PC0（Key0）和 PC2（Key1）中间，属于同一个 GPIO 口，在 CubeMX 里在同一个页面上勾选一下即可；飞线也短。配置成 `GPIO_Output`，初始电平 Low，速度 Low 就够，因为 1kHz 对 16MHz 来说不算快。这个脚只由方波功能使用，不要再分配给别的功能。

**频率**：建议 **1kHz 左右**，也就是周期 1ms。原因有两个：

- 示波器上看很稳，不用调时基
- 只有万用表时也能测：万用表在直流电压档读到的是**平均电压**，1kHz 下读数稳定，按一下降一档，比示波器还直观（3.3V × 占空比）

**步进**：学长说的 5%，技术上保守了。你的软延时最小单位大约是 0.25µs（之前标定过 `soft_delay(20000)` ≈ 5ms），周期 1ms = **4000 个单位**，1% 就是 **40 个单位 ≈ 10µs**，分辨率绰绰有余。所以 **5% 能稳稳跑通，1% 也不算激进**。我建议先按 5% 走通，想加分再改常量。改一下步长常量就能做到，成本为零。

### 二、方波怎么出来：一轮画一个完整的周期

关键设计是**每调用一次就画一个完整的“高电平加低电平”周期，然后立刻返回**，不要在里面执行死循环。

```
MODE_6() 被主循环调一次：
    PC1 = 1
    软延时(duty * 40)           ← 高电平部分
    PC1 = 0
    软延时((100 - duty) * 40)   ← 低电平部分
    返回
```

因为高电平时长加低电平时长恒等于 4000 个单位，所以**周期恒定是 1ms，只有占空比随 duty 变化**，这正是题目要的。

这样做的好处是：主循环每轮调用一次 `MODE_6()`，画完 1ms 就交还控制权，按键中断照常进入，按键的响应延迟最多 1ms，完全满足“响应要快”。如果你写成 `while(1){ 画波形 }`，主循环就再也无法处理其他事情，模式也切换不了。

一个需要接受的副作用是频率会比 1kHz 略低，因为主循环那几微秒的开销插在周期之间。示波器大概读到 990Hz，这不影响。

### 三、占空比怎么改：按键的分配与“按住不连跳”

**键位复用**，不去外接。模式 6 下：

- **KEY1 → 占空比 +5%**
- **KEY2 → 占空比 −5%**
- **KEY3 → 退出，回模式 3**

题目说“按键不够可外接”，复用符合要求，答辩时就说“模式 6 下复用 KEY1 和 KEY2 调占空比”。

**这里有一个必须处理的点**：题目明确写了“**按一下就动一档，不能按住连续跳**”。你现在的 `KEY_1` 实现是“30ms 窗口内只认一次”，**按住不放会每 30ms 触发一次，也就是连续跳**，这个写法在模式 6 下会违反评分要求。

所以模式 6 的两个调档键必须改成**“边沿触发，松开后解锁”**的写法：中断里记录一次，主循环里发现这个键已经松开，才允许下一次。

duty 用 `uint8_t`，范围 0~100，加减时进行钳位：减到 0 停住，加到 100 停住，不要回绕。答辩时这个更好解释。

### 四、模式 6 怎么进出，以及三个会冲突的地方

**进入**：把 `KEY_3` 的循环从 `3→4→5→3` 改成 `3→4→5→6→3`。还是同一个键，只多一个分支。

**冲突一：`key2_tick`**。它现在只有两个分支：`mode != 5` 时进入 able_3，`mode == 5` 时执行多击。加入模式 6 之后要变成三个分支：

- `mode == 5` → 多击
- `mode == 6` → 占空比减
- 其它 → able_3

如果不改，在模式 6 按 KEY2 会触发 3 秒倒计时。

**冲突二：`k1_flag`**。主循环里 `if (k1_flag) able_2_start();` 是无条件执行。在模式 6 下按 `+5%` 会顺便启动蜂鸣器，所以这里也要按模式进行分支。

**冲突三：`able_2` 和 `able_3` 运行期间**。它们如果正在运行，主循环会跳过模式函数，方波也就停住了。这是必然的，可以接受；倒计时或鸣叫期间方波暂停，结束之后自然会恢复。

### 五、两个必须避开的坑

**0% 和 100% 的毛刺**。如果写成“先拉高、再延时”，在 duty=0 时就会先拉高，然后立刻拉低，产生一个只有几条指令宽（约 0.1µs）的尖刺，示波器上能看到。duty=0 时**干脆不拉高**，duty=100 时**干脆不拉低**，对两端单独处理。

**乘法的位宽**。`duty * 40` 如果 duty 是 `uint8_t`、40 是 `int`，会先提升成 int 再算，所以这里安全；但如果你后面把步长改大、把周期单位数改大，记得显式用 `uint32_t` 计算，以免溢出。

### 六、怎么验证

1. 上电后按 KEY3 三下进模式 6，用示波器接 PC1 看 1kHz 方波
2. 按 KEY1，占空比每按一次加 5%（或 1%），**按住不放时应该只跳一档**
3. 到 100% 后停住，再按不变；按 KEY2 往回收
4. 没有示波器就用万用表：电压从 0V 到 3.3V，一档一档增长

---

整体只有三处改动：新增 `MODE_6()`、给 `KEY_3` 加一个分支、处理模式 6 下的两个键。等你确认这个思路后，我再逐个提供代码。

## 问题 2：按这个方案改代码——MODE_6、两个调档键、模式循环、松手解锁

**我：** 那就这样

**AI：** 红字写的就是我们讨论的方案，那就按这个方案实施。一共五处改动。

### 1. `Mycode.c` 顶部加常量和变量

放在 `mode` 那几个变量旁边：

```c
/* ===== 拓展1：GPIO 方波 ===== */
#define WAVE_STEP   5               /* 每档 5%（改成 1 就是 1% 步进） */
#define WAVE_UNIT   40              /* 1% 占用的软延时单位数 ≈ 10us */

static uint8_t duty      = 50;      /* 当前占空比 0~100 */
static volatile uint8_t k1_locked = 0;  /* KEY1 调档锁，按住只算一次 */
```

### 2. 新增 `MODE_6()`

写在哪都行，我放在 `MODE_5` 后面：

```c
void MODE_6(void)
{
	if (duty > 0)
	{
		HAL_GPIO_WritePin(GPIOC, GPIO_PIN_1, GPIO_PIN_SET);
		soft_delay((uint32_t)duty * WAVE_UNIT);
	}

	if (duty < 100)
	{
		HAL_GPIO_WritePin(GPIOC, GPIO_PIN_1, GPIO_PIN_RESET);
		soft_delay((uint32_t)(100 - duty) * WAVE_UNIT);
	}
}
```

两个 `if` 分别处理 0% 和 100% 的边界，避免出现只持续一条指令的毛刺。每一轮刚好画完一个完整周期，周期恒为 `100 * 40 = 4000` 个单位，约 1ms，也就是 1kHz。

### 3. `KEY_1` 加模式 6 分支

```c
void KEY_1(void)
{
	static uint32_t t1 = 0;

	if (mode == 6)                      /* 模式6：占空比 +5% */
	{
		if (k1_locked == 0)
		{
			k1_locked = 1;
			if (duty <= 100 - WAVE_STEP) duty += WAVE_STEP;
		}
		return;
	}

	if (HAL_GetTick() - t1 > 30)
	{
		t1 = HAL_GetTick();
		k1_flag = 1;
	}
}
```

`k1_locked` 和 `KEY_2` 现在的做法一样：中断首次进入时上锁，之后抖动或按住产生的边沿都不再响应，直到主循环看到按键松开才解锁。这就是“按住不连续跳动”。

### 4. `key2_tick` 加模式 6 分支

插在 `if (mode != 5)` 前面：

```c
	if (mode == 6)                      /* 模式6：占空比 -5% */
	{
		if (k2_cnt > 0)
		{
			k2_cnt = 0;
			if (duty >= WAVE_STEP) duty -= WAVE_STEP;
			else                   duty = 0;
		}
		return;
	}
```

`KEY_2` 中断本身已经有 `k2_locked`，而且只累加一次，所以按住不放也只会减一档。

### 5. `KEY_3` 加模式 6 循环

```c
		if      (mode == 3) mode = 4;
		else if (mode == 4) mode = 5;
		else if (mode == 5) mode = 6;
		else
		{
			mode = 3;
			HAL_GPIO_WritePin(GPIOC, GPIO_PIN_1, GPIO_PIN_RESET);   /* 换模式自动退出 */
		}
```

离开模式 6 时把 PC1 拉低，波形就停了，这就是红字里说的“换模式后自动退出”。

**`able_4` 里也补一行**，放在那三个 `WritePin` 后面：

```c
	HAL_GPIO_WritePin(GPIOC, GPIO_PIN_1, GPIO_PIN_RESET);   /* 紧急复位也停波形 */
```

### 6. `main.c` 主循环补一行

在 `key2_tick();` 下面：

```c
	  if (k1_locked && HAL_GPIO_ReadPin(GPIOC, Key1_Pin) == GPIO_PIN_RESET)
		  k1_locked = 0;              /* KEY1 松手才解锁 */
```

### 7. CubeMX 加 PC1

`Pinout` 里点 **PC1** → `GPIO_Output`，`GPIO output level = Low`，`Mode = Output Push Pull`，`Speed = Low`。生成代码就行，`USER CODE` 区不会被动到。

### 8. `Mycode.h` 加一行

```c
void MODE_6(void);
```

### 频率要标定一下

我按 `soft_delay(1) ≈ 0.25µs` 算的（之前标定过 `soft_delay(20000)` 约等于 5ms）。所以 `WAVE_UNIT = 40` 对应 1% = 10µs，周期 1ms。

烧进去用示波器量一下周期，没示波器就用万用表量平均电压：

- **周期明显不是 1ms** → 按比例改 `WAVE_UNIT`：实测 1.3ms 就改成 52

### 测法

1. KEY3 按三下进模式 6 → PC1 输出 1kHz 方波，占空比 50%
2. KEY1 按一下 +5%，**按住不放必须只跳一档**
3. 按到 100% 后不再增长，KEY2 往回减，到 0% 停住
4. 按 KEY3 → 回模式 3，PC1 变低电平，波形消失
5. 模式 6 下按 WK_UP → 回模式 1，PC1 也是低电平

## 问题 3：Duty 只改了一半、main.c 没有 case 6、k1_locked 搜不到；接 LED 能不能看占空比

**我：** 之前定义过duty了，所以我把mode6相关的duty改成了Duty,不知道改完了没有，可能会漏，你看看。还有，主函数搜索不到k1-locked变量，报错了，然后我现在不在实验室，看不到波形啥的，我直接给pc1上接个led能不能看到亮度变化

**AI：** 我查完了，漏了三处，而且会直接导致编译失败。

### 一、`Duty` 只改了一半

`Mycode.c:125` 和 `:131` 还是小写 `duty`：

```c
125:		soft_delay((uint32_t)duty * WAVE_UNIT);
131:		soft_delay((uint32_t)(100 - duty) * WAVE_UNIT);
```

现在的 `duty` 是 `MODE_3`（第 59 行）和 `MODE_5`（第 99 行）各自函数里的 `static`，**函数外面看不见**。所以这两行会报 `duty: undeclared identifier`。改成 `Duty`：

```c
		soft_delay((uint32_t)Duty * WAVE_UNIT);
...
		soft_delay((uint32_t)(100 - Duty) * WAVE_UNIT);
```

### 二、`main.c` 的 switch 里没有 `case 6`

你 `main.c:121-128` 只有 1~5：

```c
			  case 5: MODE_5(); break;
```

加一行：

```c
			  case 5: MODE_5(); break;
			  case 6: MODE_6(); break;
```

不加这行，进了模式 6 主循环什么都不画，PC1 一直是低电平。

### 三、`k1_locked` 报错的原因

`Mycode.c:18` 是 `static volatile uint8_t k1_locked`，`static` 的意思是“只在本文件可见”，`main.c` 当然搜不到。

**把 `main.c:104-105` 那两行删掉**，挪到 `Mycode.c` 的 `key2_tick()` 开头（它每轮都跑），顺便把“离开模式 6 停波形”也补上：

```c
void key2_tick(void)
{
	/* 松开才解锁 */
	if (k2_locked && HAL_GPIO_ReadPin(GPIOF, Key2_Pin) == GPIO_PIN_RESET)
		k2_locked = 0;

	if (k1_locked && HAL_GPIO_ReadPin(GPIOC, Key1_Pin) == GPIO_PIN_RESET)
		k1_locked = 0;

	if (mode != 6)
		HAL_GPIO_WritePin(GPIOC, GPIO_PIN_1, GPIO_PIN_RESET);   /* 离开模式6，波形停 */

	if (mode == 6)                      /* 模式6：占空比 -5% */
	{
		...
```

第三行是我发现的一个漏洞：`KEY_0` 在模式 6 下也能按（它会切到模式 1/2），但 `KEY_0` 里没有拉低 PC1，所以如果占空比正好是 100%，切走后 PC1 会一直保持高电平。加上这一行，“任何方式离开模式 6”都会自动停波形，比在 `KEY_3` 里单独写更保险。

### 四、接 LED 看亮度——能，但只能看两头

**能看到的**：

- 占空比 0% → 灭
- 100% → 全亮
- 中间 → 半亮左右，但**5% 一档的差别肉眼绝对分不出来**

两个原因：1kHz 远超人眼闪烁融合频率，你看到的只是平均亮度；而且 LED 有导通阈值，亮度对占空比是非线性的（50% 看起来不止半亮）。

**没示波器时更靠谱的三种验证手段**：

1. **万用表直流电压档量 PC1 对地**（最准）。3.3V 供电时：50% ≈ 1.65V，每按一下 ±0.165V，0% ≈ 0V，100% ≈ 3.3V。数字万用表能直接读出这个变化，比眼睛可靠得多。

2. **临时把步进放大**：`#define WAVE_STEP 25`，变成 0/25/50/75/100 五档，肉眼一看就知道亮度在变、而且“按住不连跳”。**验证完记得改回 5**，答辩时步进越细得分越高。

3. **用 LED 验证“按住不连跳”**：按住 KEY1 三秒，亮度只能变一档。这条用眼睛就能判断，是拓展 1 的评分点之一。

## 问题 4：改完灯不亮——先把「没进模式 6」和「引脚不通」分开

**我：** 改完了，灯不亮，步进改25了，点3-4次都不亮

**AI：** 代码这边我核对过了，三处都改对了：`gpio.c:78-83` PC1 配成输出推挽、初始低，`main.c:127` 有 `case 6: MODE_6()`，`Mycode.c:125/131` 的 `Duty` 大小写已修，所以配置本身已经能输出波形。现在剩下两种可能：**没进入模式 6**，或者 **PC1 到 LED 这一段不通**。用一个不依赖按键的测试把两者分开——在 `MODE_1` 的 `phase ^= 1;` 下面加一行 `HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_1);`，上电就该每秒闪一次；判断是否进了模式 6 则看 LED0：模式 3/4/5 里 LED0 都在呼吸，模式 6 只操作 PC1，所以 LED0 会**停止变化并保持当前亮度**，这是最明显的标志。

## 问题 5：确定是引脚的问题，换脚只改两个宏

**我：** 第一步试了，闪的快了，但pc1还是不行，第二步也试了，发现led0可以，应该就是pc1的问题了

**AI：** 那就确定了：**程序、方波逻辑、`soft_delay` 标定全都正常，出问题的是 PC1 这个引脚**。换引脚，先加两个宏，这样以后换脚只改两行。

```c
/* ===== 拓展1：GPIO 方波 ===== */
#define WAVE_STEP   25              /* 步进百分比，测完改回 5 */
#define WAVE_UNIT   40              /* 1% 的软延时单位数 */
#define WAVE_PORT   GPIOC           /* 波形输出端口 */
#define WAVE_PIN    GPIO_PIN_3      /* 波形输出引脚，换脚只改这两行 */
```

然后把文件里那五处 `GPIOC, GPIO_PIN_1` 全换成 `WAVE_PORT, WAVE_PIN`（`MODE_6` 两处、`KEY_3` 一处、`KEY_UP` 一处）。CubeMX 里点 **PC3** → `GPIO_Output`，`GPIO output level = Low`，`Mode = Output Push Pull`，`Speed = Low`，再把 **PC1 取消掉**（右键 → `Reset_State`），否则 PC1 还占着。生成代码只会更新 `main.c`、`gpio.c` 这些由 CubeMX 管理的文件，`Mycode.c` 不受影响。

## 问题 6：按住 KEY1/KEY2 才微亮——接法不对，加上按一下就掉一档的连减 bug

**我：** 来来回回又改了几次，硬件也测了好多次。说一下状况：首先，硬件没有问题，灯，引脚，杜邦线都没有问题，也没接错。然后我目前的代码（你自己再去看一下）进入mode6后，一直按着key1/2,新接的led里面的灯珠才会微亮，接法是长接引脚，短接3.3v(其他接法都不会微亮)，所以肯定是你的代码问题

**AI：** 我把现在的代码全部看了一遍。代码里确实有 bug，但**先说接法：你现在的接法在物理上就不会亮**。

### 一、你的接法有问题

「长脚（阳极）接引脚，短脚（阴极）接 3.3V」这个接法，当引脚输出高电平 3.3V 时，LED 两端都是 3.3V，**压差为 0**，因此没有电流。当引脚输出低电平时，LED 反偏，也没有电流。所以无论引脚输出什么，都不会亮。你看到的「微亮」是**反向漏电流**（几微安），说明此刻引脚处于**高阻或悬空**状态，也就是根本没有输出，而不是代码在输出。

正确接法只有两种：

- **高电平点亮**：长脚 → 引脚，短脚 → **330Ω 电阻 → GND**
- **低电平点亮**：长脚 → **3.3V**，短脚 → 引脚

**必须串电阻**，直接接 3.3V 会烧。

### 二、`KEY_3:334` 和 `KEY_UP:341` 还是 PC1

```c
		HAL_GPIO_WritePin(GPIOC, GPIO_PIN_1, GPIO_PIN_RESET);   /* 334 行 */
	HAL_GPIO_WritePin(GPIOC, GPIO_PIN_1, GPIO_PIN_RESET);       /* 341 行 */
```

这两行没有改成 `GPIO_PIN_3`，退出模式 6 时 PC3 会停在最后那个电平上。

### 三、这就是「按住 KEY1 才微亮」的原因

`key2_tick` 每轮循环都会取用一次 `k2_cnt`：

```c
	if (mode == 6)
	{
		if (k2_cnt > 0)
		{
			k2_cnt = 0;
			if (Duty >= WAVE_STEP) Duty -= WAVE_STEP;   /* 每轮都可能减一次 */
```

`Duty` 初值是 50，`WAVE_STEP` 是 25，所以**按 KEY2 两次就是 0%**，灯全灭。然后你按住 KEY1，`Duty` 从 0 加一档变成 25，**25% 占空比看起来就是「微亮」**。

你描述的「按住才微亮」和这个完全对得上：不是按住才输出，而是按住才把占空比从 0 加到 25。

修法是把减档逻辑搬进中断，和 KEY_1 对称，主循环不再每轮扫描：

```c
void KEY_2(void)
{
	if (k2_locked == 0)
	{
		k2_locked = 1;
		k2_cnt++;
		k2_last = HAL_GetTick();

		if (mode == 6)                      /* 模式6：减一档 */
		{
			if (Duty >= WAVE_STEP) Duty -= WAVE_STEP;
			else                   Duty = 0;
		}
	}
}
```

`key2_tick` 里的模式 6 分支改成只清理，不处理：

```c
	if (mode == 6)
	{
		k2_cnt = 0;
		return;
	}
```

## 问题 7：重读烧录里的代码，用固定 50% 的版本绕开所有变量

**我：** 你读的是我上一轮没保存的代码，不是我现在烧录到代码，重读

**AI：** 我重读了正在烧录的代码（346 行版本）。先确认：现在 PC3 相关的五处都改对了，`MODE_1` 里那段自检也删干净了，`KEY_3:328`、`KEY_UP:335` 都是 PC3。`MODE_6` 每次输出高电平和低电平各 2000 个延时单位（`50 × 40`），占空比 50%，频率约 1 kHz。**从逻辑上看，这段代码输出的就是 50% 方波，没有错。**

但「按住 KEY1 或 KEY2 才微亮」这个现象，正好指向另外两个问题同时存在。

### 一、你的接法，理论上不可能亮

「长脚接引脚，短脚接 3.3V」这个接法，当 PC3 输出高电平 3.3V 时：

```
PC3 (3.3V) ──[LED]── 3.3V
```

LED 两端都是 3.3V，**压差 0V，没有电流**。当 PC3 输出低电平 0V 时，LED 反偏，也没有电流。

**唯一能“微亮”的解释是：你接的那个“3.3V”其实是 5V。** 5V − 3.3V = 1.7V，正好在 LED 的开启电压附近，只能漏过去一丝电流，所以你看到的是微亮，而不是亮。

### 二、`Duty` 被按到 0 了

`Mycode.c:17` 初值是 50，`WAVE_STEP` 是 25：

- 按 **KEY2 两次**：50 → 25 → **0**
- `Duty = 0` 时，`MODE_6` 里 `if (Duty > 0)` 不成立，引脚**一直是低电平**
- 这时候按住 **KEY1** → `Duty` 从 0 加一档变 25 → 引脚有 25% 时间在高电平 → 你看到「微亮」

**你描述的「按住才微亮」和这个完全吻合**：并不是按住才输出，而是按住后占空比才从 0 变成 25。

### 三、先做这个测试，绕开所有变量

把 `MODE_6` 整个换成固定方波，不经过 `Duty`：

```c
void MODE_6(void)
{
	HAL_GPIO_WritePin(GPIOC, GPIO_PIN_3, GPIO_PIN_SET);
	soft_delay(2000);
	HAL_GPIO_WritePin(GPIOC, GPIO_PIN_3, GPIO_PIN_RESET);
	soft_delay(2000);
}
```

进模式 6 **一下键都别按**，看 LED：

| 现象 | 结论 |
|---|---|
| 稳定半亮 | 引脚、代码全没问题，之前就是 `Duty` 被按到 0 + 接法不对 |
| 微亮 | 引脚在输出，但接法让电流极小，99% 是那个“3.3V”其实是 5V |
| 完全不亮 | 万用表量 PC3 对地电压，读数告诉我 |

### 四、万用表这一下最关键

模式 6 下量 PC3 对地：

- **1.65V 左右** → 50% 方波完全正常，问题 100% 在 LED 那一段
- **3.3V 恒定** → `Duty` 已经是 100，引脚一直高
- **0V** → `Duty` 是 0，引脚一直低
- **3.3V 和 0V 之间乱跳** → 正常方波，但周期不对

我敢肯定，如果电压在 1.65V 左右，问题就只剩 LED 接线。代码这边我已经逐行核对，`MODE_6` 输出的就是一个标准 50% 方波。

## 问题 8：接法的最终结论

**我：** 改了，还是不亮……（略）

**AI：** 照板载 LED 的接法接，你这个接法本身是对的，**但外接必须串电阻，板载的电阻焊在板子上了，你的没有**——`3.3V ──[LED 长脚 +]──[LED 短脚 −]──[330Ω]── PC3`，这样 PC3 输出低时亮，输出高时灭，跟 LED0、LED1 一样。另外提醒一句：`gpio.c:57` 现在把 PC3 上电初始化成**高电平**，所以**上电时 PC3 是高，低电平点亮的接法下就是灭的**，只有真正跑进 `MODE_6` 且 `Duty < 100` 时才会有低电平出现。

## 问题 9：真正的原因不是电阻，是面包板电源轨分段供电

**我：** 硬件bug:面包板上的3.3和gnd那一条线上，有一些间断处，我开始把单片机的3.3和-级接到了板子最边缘，而每隔一个间断处他的"电性"就会减弱，导致不能正常响应，当时key3离接线处最近，所以不会丢失，而其他按键离得很远，导致会偶尔丢失，而蜂鸣器离的最远，完全"丢电"导致不响。现在均已解决

**AI：** 这就说得通了，而且比"引脚烧了"合理得多。

面包板的红、蓝两条电源轨本身是**分段断开**的，段与段之间不导通。你把 3.3V 和 GND 接在板子最边上，中间靠段与段之间夹着的那点接触电阻勉强导通，越往远走电压掉得越厉害。KEY3 离得近，电压够，所以从不丢；其他按键离得远，临界电压下就偶尔丢；蜂鸣器最远，电压低到不工作，所以完全不响。

**我之前说 PC1、PC3 很可能是被 LED 的电流打坏了，这个判断作废。** 那两个脚很可能一直是正常的，是那个灯所在的段没电。PC3 现在先别下定论，等拓展1 重做时再试。

## 问题 10：呼吸灯用了 `HAL_GetTick()`，而题目禁止用定时器（含 SysTick）

**我：** 我看了一下代码，你那个呼吸灯，也就是mode34用了gettick把，但题目要求不让用

**AI：** 你说得对，这是违规的。`MODE_3`、`MODE_4` 里的 `HAL_GetTick()` 最终读取的是 SysTick 计数，而题目明确禁止呼吸灯和方波使用任何定时器；方波 `MODE_6` 本来就是纯软延时，没有问题，违规的是三个呼吸函数。

### 关键点

`soft_delay(duty*400)` 加 `soft_delay((100-duty)*400)` 合计固定 **40000** 个单位，大约就是 **10ms**，也就是一次 PWM 周期。所以**调用这个函数一次，本身就会耗费 10ms**，根本不需要另外读取时间来判断"过了 10ms 没有"。因此把那层 `HAL_GetTick()` 判断去掉，改成每调用一次就让 `duty` 走一档即可。

### `MODE_3` 整个替换

```c
void MODE_3(void)
{
	static int16_t duty = 0;
	static int8_t  dir  = 1;

	duty = (int16_t)(duty + dir);
	if (duty >= 100) { duty = 100; dir = -1; }
	if (duty <= 0)   { duty = 0;   dir =  1; }

	HAL_GPIO_WritePin(GPIOF, LED1_Pin, GPIO_PIN_SET);

	HAL_GPIO_WritePin(GPIOF, LED0_Pin, GPIO_PIN_SET);
	soft_delay((uint32_t)duty * 400);
	HAL_GPIO_WritePin(GPIOF, LED0_Pin, GPIO_PIN_RESET);
	soft_delay((uint32_t)(100 - duty) * 400);
}
```

### `MODE_4` 整个替换

```c
void MODE_4(void)
{
	HAL_GPIO_WritePin(GPIOF, LED0_Pin, GPIO_PIN_SET);

	duty = (int16_t)(duty + dir);
	if (duty >= 100) { duty = 100; dir = -1; }
	if (duty <= 0)   { duty = 0;   dir =  1; }

	HAL_GPIO_WritePin(GPIOF, LED1_Pin, GPIO_PIN_SET);
	soft_delay((uint32_t)duty * 400);
	HAL_GPIO_WritePin(GPIOF, LED1_Pin, GPIO_PIN_RESET);
	soft_delay((uint32_t)(100 - duty) * 400);
}
```

### `MODE_5` 整个替换

```c
void MODE_5(void)
{
	static int16_t duty = 0;
	static int8_t  dir  = 1;

	duty += dir;
	if (duty >= 100) { duty = 100; dir = -1; }
	if (duty <= 0)   { duty = 0;   dir =  1; }

	HAL_GPIO_WritePin(GPIOF, LED0_Pin, GPIO_PIN_RESET);
	soft_delay((uint32_t)duty * 400);
	HAL_GPIO_WritePin(GPIOF, LED0_Pin, GPIO_PIN_SET);
	soft_delay((uint32_t)(100 - duty) * 400);

	if (!able_3_busy())
	{
		HAL_GPIO_WritePin(GPIOF, LED1_Pin, GPIO_PIN_SET);
	}
}
```

### 删掉一行

`MODE_3` 上面那三个文件级变量里的 `step` 不再需要了：

```c
	static uint32_t step = 0;      /* 删掉这一行 */
```

`duty` 和 `dir` 要保留，`MODE_3` 和 `MODE_4` 共用。

---

改完之后，`MODE_3`、`MODE_4`、`MODE_5` 里连一个 `HAL_GetTick()` 都没有了，全是 `soft_delay`。节奏仍然是 **10ms 一档、100 档、一个来回 1 秒**，肉眼看到的呼吸速度和现在一样。

答辩时可以这么讲：**"呼吸的节奏不是靠定时器打拍子的，而是靠 PWM 周期本身的耗时。一个周期 10ms，跑完一个周期就把占空比挪 1%，跑 100 个周期就是 1 秒，完全由循环本身的执行时间决定。"**

另外，`MODE_1`、`MODE_2` 里的 `HAL_GetTick()` 不用改，题目只对呼吸灯和方波限制了定时器，闪烁那两条是允许使用的；`able_3` 的 3 秒倒计时也允许，只有基本5 和拓展1 有"不得用定时器"的限制。

## 问题 11：改完之后会不会影响按键响应

**我：** 改了之后会不会对按键造成影响，中断不响应啥的

**AI：** 不会有影响。原因有三点：

**一、中断和软延时是并行的**

按键走的是 EXTI 硬件中断，优先级 0。`soft_delay` 只是在死等一个 `volatile` 计数器，CPU 该被中断打断照样会被打断。**你按下去的那一刻中断就进去了**，跟主循环卡在哪个延时里无关。真会"中断不响应"的情况只有一种：中断里自己调 `HAL_Delay`（SysTick 优先级 15 比 EXTI 低，会死等），这个坑你早就避开了。

**二、一帧的耗时没变**

改之前是 `soft_delay(duty*400) + soft_delay((100-duty)*400)` 加上一句 `HAL_GetTick()`；改之后是同样的两段延时。40000 个单位还是 10ms 左右，所以按键响应的最坏延迟还是 **10ms 量级**，跟改之前完全一样。

**三、变的只是"怎么数时间"**

原来是"每 10ms 查一次表"，现在是"反正这一圈就是 10ms，走完就挪一档"。`HAL_GetTick()` 那句从头到尾没碰过中断，删掉它也不会碰到。

---

### 改完顺手测这三个

1. **模式3 呼吸中按 KEY3**：能立刻切到模式4（最多延迟 10ms，感觉不到）
2. **模式5 呼吸中连打两下**：双击能识别、呼吸不停
3. **任意模式按 WK_UP**：立刻回模式1

如果这三个都正常，就说明中断链路完好。

只有一种情况需要注意：主循环一帧 10ms，意味着 `key2_tick` 里的"松开解锁"也是每 10ms 才看一次引脚。按键松手时间短于 10ms 才可能漏解锁，人手做不到这么快，不用管。

---
