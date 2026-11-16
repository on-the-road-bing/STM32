# TIM 定时器

>**TIM 与 SysTick**
>
>- SysTick：隶属于 ARM Cortex-M 内核，是所有基于该内核的芯片（无论哪家厂商）都标配的组件。它的存在目的非常纯粹，主要是为了给 RTOS 或简单的裸机程序提供一个标准的、可移植的系统时基。
>- TIM (Timer)：是 STM32 等芯片厂商在芯片内部集成的外设。数量众多（如 STM32F4 系列多达 14 个），并且被划分为基本、通用、高级等不同类型，以满足从简单定时到复杂电机控制的各种需求。

## 一、TIM 概述

STM32 微控制器中的定时器（Timer）是一个非常重要的外设模块，用于生成精确的时间延迟、测量时间间隔、生成PWM信号以及执行其他与时间相关的任务。定时器在嵌入式系统中应用广泛，例如控制LED闪烁、驱动电机、测量传感器信号等。

- TIM（Timer）定时器

- 对输入的时钟进行计数，并在计数值达到设定值时触发中断。

- 分为基本定时器、通用定时器、高级定时器三种类型。

- 不仅具备基本的定时中断功能，而且还包含内外时钟源选择、输入捕获、输出比较、编码器接口、主从触发模式等多种功能。

  <img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image-20261025142747153.png" alt="image-20261025142747153" style="zoom:50%;" />

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image-20260813160224862.png" alt="image-20260813160224862" style="zoom:50%;" />

- 基本定时器：只会“数数”，用于内部定时。
- 通用定时器：能“数数”也能“看”（输入捕获）和“说”（PWM输出），是通用接口。
- 高级定时器：在通用基础上，专精于安全、精确地控制功率器件（电机、电源）。

### STM32定时器的组成

1. 计数器（Counter）：定时器的核心部分，用于计数。计数器可以是向上计数、向下计数或双向计数。
2. 预分频器（PSC）：用于将输入时钟频率分频，从而调整计数器的计数速度。
3. 自动重装载寄存器（ARR）：用于设置计数器的最大值。当计数器达到该值时，会触发溢出事件，并重新开始计数。
4. 捕获/比较寄存器（CCR）：用于捕获外部事件的时间或生成PWM信号。

### 定时器用途

1. 定时器：用于生成精确的时间延迟。
2. 输入捕获：用于测量外部信号的脉冲宽度或频率。
3. 输出比较：用于生成PWM信号或触发其他事件。
4. PWM输出：用于生成脉宽调制信号，常用于控制电机或LED亮度。

## 二、基本定时器

基本定时器是STM32定时器家族中最简单的一种，通常用于简单的计时任务。

### 基本定时器工作原理

STM32的基本定时器通常由以下几个部分组成：

1. 计数器（Counter）：一个16位的向上计数器，用于计数时钟脉冲。
2. 预分频器（Prescaler）：用于将输入时钟频率分频，从而降低计数器的计数速度。
3. 自动重装载寄存器（Auto-reload Register）：用于设置计数器的上限值，当计数器达到该值时，会自动重置并触发中断或事件。

基本定时器的工作流程如下：

1. 定时器从0开始计数，每接收到一个时钟脉冲，计数器加1。
2. 当计数器达到自动重装载寄存器中设置的值时，计数器重置为0，并触发中断或事件。
3. 通过调整预分频器和自动重装载寄存器的值，可以控制定时器的计数周期和触发频率。

### 基本定时器功能框图

定时器的核心是时基，不仅基本定时器有，通用定时器和高级定时器也有。

> **"时基单元"**（所有定时器都有）：时钟源 → PSC 预分频 → CNT 计数器 → ARR 自动重装载 → 溢出产生更新事件（UEV）；
>
> 定时器工作频率 = 外设总线频率 / (PSC + 1)
> 定时频率 = 定时器工作频率 / (CNT + 1)

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image313.png" alt="图 30‑2 基本定时器功能框图" style="zoom: 50%;" />



- 时钟源

  定时器时钟 TIMxCLK，即内部时钟 CK_INT，是经 APB1 预分频器后分频提供。

  如果APB1 预分频系数为 1，则频率不变，否则频率乘以 2。库函数中APB1 预分频的系数是2，即 PCLK1=36M，所以定时器时钟 TIMxCLK=36*2=72M。

  >- 定时器时钟倍频规则：STM32 硬件检测到 APB1 分频系数不是 1，就自动把送给定时器的时钟再乘 2。
  >
  >为什么这样设计？因为 APB1 是低速外设总线，很多外设不需要 72M；但定时器需要较高计数频率来提高分辨率、捕获精度和 PWM 精度，所以给定时器单独倍频。
  >
  >- 为什么定时器时钟不是定时器的频率？
  >
  >定时器时钟是输入给定时器的基准时钟，而“定时器频率”通常指定时器经过预分频、计数、重装载之后实际的工作节奏。

- 计数器时钟

  定时器时钟经过 PSC 预分频器之后，即计数器时钟 CK_CNT，用来驱动计数器计数。具体计算方式为：CK_CNT = TIMxCLK / (PSC+1)。

- 计数器

  计数器是一个 16 位的计数器，只能往上计数 。当计数达到自动重装载寄存器的值时产生更新事件，并清零从头开始计数。

- 自动重装载寄存器

  自动重装载寄存器 ARR 是一个 16 位的寄存器，里面装着计数器能计数的最大数值。当计数到这个值的时候，如果使能了中断的话，定时器就产生溢出中断。

- 定时时间计算

  定时器的定时时间等于计数器的中断周期乘以中断的次数。计数器在 CK_CNT 的驱动下，计一个数的时间则是 CK_CNT 的倒数，等于：1 /（TIMxCLK / (PSC + 1)），产生一次中断的时间则等于：(1 / CK_CNT) * (ARR + 1)。如果在中断服务程序里面设置一个变量 time，用来记录中断的次数，那么就可以计算出我们需要的定时时间等于：(1 / CK_CNT) * (ARR + 1)  *  time。

**定时周期速查**

| 目标周期 | Prescaler | Period | 说明                       |
| :------- | :-------- | :----- | :------------------------- |
| 1 ms     | 71        | 999    | 72MHz / 72 / 1000 = 1kHz   |
| 10 ms    | 719       | 999    | MHz / 720 / 1000 = 100Hz   |
| 100 ms   | 7199      | 999    | 72MHz / 7200 / 1000 = 10Hz |
| 1 s      | 7199      | 9999   | 72MHz / 7200 / 10000 = 1Hz |



### 基本定时器配置

```c
//定时器基本初始化结构体
typedef struct {
    uint32_t Prescaler;          // PSC预分频器设置
    uint32_t CounterMode;        // 定时器计数方式，可为向上计数、向下计数以及中心对齐模式
    uint32_t Period;             // 定时器周期，即设定自动重载寄存器的值，0~65535。
    uint32_t ClockDivision;      // 时钟分频，设置定时器时钟CK_INT频率与数字滤波器采样时钟频率分频比，基本定时器无此功能。
    uint32_t RepetitionCounter;  // 重复计算器，高级控制寄存器专用寄存器位，利用它可以控制输出PWM的个数。基本定时器无此功能。
} TIM_BaseInitTypeDef;


// 管理TIM外设的句柄（TIM_HandleTypeDef）。
typedef struct
{
    TIM_TypeDef                 *Instance;     // 寄存器基地址
    TIM_Base_InitTypeDef        Init;          // 定时器时基所需参数
    HAL_TIM_ActiveChannel       Channel;       // 当前激活的通道
    DMA_HandleTypeDef           *hdma[7U];     // DMA句柄数组
    HAL_LockTypeDef             Lock;          // 锁定对象（用于互斥）
    __IO HAL_TIM_StateTypeDef   State;         // 定时器操作状态
} TIM_HandleTypeDef;

//函数
HAL_StatusTypeDef HAL_TIM_Base_Init(TIM_HandleTypeDef *htim);

void HAL_TIM_Base_MspInit(TIM_HandleTypeDef *htim);	

HAL_StatusTypeDef HAL_TIM_Base_Start_IT(TIM_HandleTypeDef *htim);


/**
 * HAL_TIM_Base_MspInit()讲解
 * HAL库中定时器初始化的底层硬件配置回调函数，负责配置与具体 MCU 硬件强相关的部分，将外设的功能逻辑与底层硬件初始化分离
 * 调用关系：HAL_TIM_Base_MspInit() 由高层初始化函数 HAL_TIM_Base_Init() 自动调用
 * 主要职责：
 * 这个函数主要处理三项硬件初始化工作：
 * 1.使能外设时钟：开启对应定时器的总线时钟（如 __HAL_RCC_TIM2_CLK_ENABLE()）。
 * 2.配置中断 (NVIC)：若使用中断，则在此设置中断优先级并使能。
 * 3.配置GPIO：若定时器通道用作PWM输出或输入捕获，则在此初始化相关I/O引脚为复用功能
 
 * 独立设计好处：
 * 1.代码模块化：代码更清晰，易于维护。HAL_TIM_Base_Init()负责抽象的定时器参数（如分频系数、计数模式），而HAL_TIM_Base_MspInit() 负责具体的MCU硬件实现。
 * 2.便于跨芯片移植：更换同系列不同型号的MCU时，通常只需修改 HAL_TIM_Base_MspInit() 等Msp文件，高层逻辑代码可保持不变。
 * 3.灵活性高：该函数在HAL库中有一个弱定义 (weak)，你可以在自己的代码中重写（覆盖） 它，实现自定义的底层初始化，而无需修改HAL库源码。
*/
```

### 实验：基本定时器定时

#### 硬件设计

本实验利用基本定时器 TIM6/7 定时 1s，1s 时间到 LED 翻转一次。基本定时器是单片机内部的资源，没有外部 IO，不需要接外部电路，现只需要一个 LED 即可。

#### 软件设计

> 1. 使能定时器时钟 TIMx_CLK, x[6,7]；  __TIMX_CLK_ENABLE()
> 2. 初始化时基初始化结构体；                HAL_TIM_Base_Init()
> 3. 使能 TIMx, x[6,7] update 中断；
> 4. 启动定时器；
> 5. 编写中断服务程序

1. 基本定时器宏定义

> 使用宏定义非常方便程序升级、移植。

```C
#define BASIC_TIMx_PRESCALER		71 
#define BASIC_TIMx_PERIOD			1000 

#define BASIC_TIM TIM6
#define BASIC_TIM_CLK_ENABLE()		__TIM6_CLK_ENABLE()

#define BASIC_TIM_IRQ				TIM6_IRQn
#define BASIC_TIM_INT_FUN			TIM6_IRQHandler
```

2. 基本定时器配置

```C
/* 句柄定义 */
TIM_HandleTypeDef htimx;

void BASIC_TIMx_Init(void)
{
    htimx.Instance = BASIC_TIMx;
    htimx.Init.Prescaler = BASIC_TIMx_PRESCALER;
    htimx.Init.CounterMode = TIM_COUNTERMODE_UP;
    htimx.Init.Period = BASIC_TIMx_PERIOD;
    HAL_TIM_Base_Init(&htimx); //初始化	
    HAL_TIM_Base_Start_IT(&htimx); // 启动定时器中断
}


// TIM6底层硬件初始化（HAL回调）
void HAL_TIM_Base_MspInit(TIM_HandleTypeDef *htim)
{
    if (htim->Instance == TIM6) {
        __HAL_RCC_TIMx_CLK_ENABLE();	// 使能TIM6时钟
         // 配置中断优先级（抢占优先级1，子优先级0）
		HAL_NVIC_SetPriority(BASIC_TIM_IRQ, 1, 0);
		HAL_NVIC_EnableIRQ(BASIC_TIM_IRQ);
    }
}
```

3. 定时器中断优先级配置

```C
/* 外设中断配置 */
HAL_NVIC_SetPriority(BASIC_TIM_IRQ, 1, 0);
HAL_NVIC_EnableIRQ(BASIC_TIM_IRQ);
```

4. 定时器中断服务程序

   在中断服务程序的最后，要把相应的中断标志位清除掉，切记。

```C
void TIM6_IRQHandler(void)
{
    HAL_TIM_IRQHandler(&TIM_TimeBaseStructure);
}
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    time++;
}
```

主函数

> LED1 以 1s 的频率闪烁一次。

```c
int main(void)
{   
    SystemClock_Config();			/* 配置系统时钟为72MHz */
    LED_GPIO_Config();
    BASIC_TIMx_Init();      		/* 基本定时器初始化：1ms中断一次，启动定时器中断 */
    while(1) {
        if ( time == 1000 ) { /* 1000 * 1 ms = 1s 时间到 */
            time = 0;   
            LED1_TOGGLE; 	/* LED1 取反 */      
        }        
    }
}
```

## 三、通用定时器

通用定时器的核心是一个16位或32位的计数器，它可以根据配置向上计数、向下计数或双向计数。定时器的时钟源可以来自内部时钟（如APB总线时钟）或外部时钟。通用定时器广泛应用于嵌入式系统中，例如电机控制、LED调光、信号捕获等场景。

STM32的通用定时器通常由以下几个部分组成：

- 计数器：用于计数时钟脉冲。
- 预分频器：用于降低输入时钟频率。
- 自动重装载寄存器：用于设置计数器的最大值。
- 捕获/比较寄存器：用于捕获输入信号或比较输出信号。

**计数器模式**

1. 向上计数模式：计数器从0开始递增，直到达到自动重装载寄存器的值，然后重新从0开始计数。
2. 向下计数模式：计数器从自动重装载寄存器的值开始递减，直到达到0，然后重新从自动重装载寄存器的值开始计数。
3. 中心对齐模式：计数器先向上计数到自动重装载寄存器的值，然后向下计数到0，如此循环。

## 四、高级定时器

高级定时器的核心是一个16位或32位的计数器，可以通过配置预分频器和自动重装载寄存器来控制计数器的频率和周期。高级定时器通常包含多个捕获/比较通道，用于生成PWM信号或测量输入信号的频率和占空比。

高级定时器（Advanced Timer）是一种功能强大的外设，广泛应用于需要精确时间控制的场景，如电机控制、PWM信号生成、编码器接口等。与基本定时器和通用定时器相比，高级定时器提供了更多的功能和灵活性，能够满足复杂的应用需求。

高级定时器通常具有以下特点：

- 支持多种计数模式（向上、向下、中央对齐）。
- 支持多通道PWM输出。
- 支持死区时间插入，适用于电机控制。
- 支持编码器接口。
- 支持事件触发和中断。



