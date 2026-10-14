# 一、TIM 定时器

## TIM 简介

- TIM（Timer）定时器
- 对输入的时钟进行计数，并在计数值达到设定值时触发中断。
- 分为基本定时器、通用定时器、高级定时器三种类型。
- 不仅具备基本的定时中断功能，而且还包含内外时钟源选择、输入捕获、输出比较、编码器接口、主从触发模式等多种功能

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image-20260813160224862.png" alt="image-20260813160224862" style="zoom:50%;" />

# 二、基本定时器

## 基本定时器功能框图

定时器的核心是时基，不仅基本定时器有，通用定时器和高级定时器也有。

> **"时基单元"**（所有定时器都有）：时钟源 → PSC 预分频 → CNT 计数器 → ARR 自动重装载 → 溢出产生更新事件（UEV）；
>
> 定时器工作频率 = 外设总线频率 / (PSC + 1)
> 定时频率 = 定时器工作频率 / (CNT + 1)

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image313.png" alt="图 30‑2 基本定时器功能框图" style="zoom: 50%;" />



- 时钟源

  定时器时钟 TIMxCLK，即内部时钟 CK_INT，是经 APB1 预分频器后分频提供。

  如果APB1 预分频系数为 1，则频率不变，否则频率乘以 2。库函数中APB1 预分频的系数是2，即 PCLK1=36M，所以定时器时钟 TIMxCLK=36*2=72M。

- 计数器时钟

  定时器时钟经过 PSC 预分频器之后，即计数器时钟 CK_CNT，用来驱动计数器计数。PSC 是一个16 位的预分频器，可以对定时器时钟 TIMxCLK 进行 1~65536 之间的任何一个数进行分频。具体计算方式为：CK_CNT=TIMxCLK/(PSC+1)。

- 计数器

  计数器是一个 16 位的计数器，只能往上计数，最大为2^12 - 1 。当计数达到自动重装载寄存器的值时产生更新事件，并清零从头开始计数。

- 自动重装载寄存器

  自动重装载寄存器 ARR 是一个 16 位的寄存器，里面装着计数器能计数的最大数值。当计数到这个值的时候，如果使能了中断的话，定时器就产生溢出中断。

- 定时时间计算

  定时器的定时时间等于计数器的中断周期乘以中断的次数。计数器在 CK_CNT 的驱动下，计一个数的时间则是 CK_CNT 的倒数，等于：1/（TIMxCLK/(PSC+1)），产生一次中断的时间则等于：(1/CK_CNT) * (ARR+1)。如果在中断服务程序里面设置一个变量 time，用来记录中断的次数，那么就可以计算出我们需要的定时时间等于：(1/CK_CNT) * (ARR+1) * time。

**定时周期速查**

| 目标周期 | Prescaler | Period | 说明                       |
| :------- | :-------- | :----- | :------------------------- |
| 1 ms     | 71        | 999    | 72MHz / 72 / 1000 = 1kHz   |
| 10 ms    | 719       | 999    | MHz / 720 / 1000 = 100Hz   |
| 100 ms   | 7199      | 999    | 72MHz / 7200 / 1000 = 10Hz |
| 1 s      | 7199      | 9999   | 72MHz / 7200 / 10000 = 1Hz |



## 基本定时器相关结构体与函数

```c
//定时器基本初始化结构体
typedef struct {
    uint32_t Prescaler;          // PSC预分频器设置，0~65535。
    uint32_t CounterMode;        // 定时器计数方式，可为向上计数、向下计数以及中心对齐模式
    uint32_t Period;             // 定时器周期，即设定自动重载寄存器的值，0~65535。
    uint32_t ClockDivision;      // 时钟分频，设置定时器时钟CK_INT频率与数字滤波器采样时钟频率分频比，基本定时器无此功能。
    uint32_t RepetitionCounter;  // 重复计算器，高级控制寄存器专用寄存器位，利用它可以控制输出PWM的个数。基本定时器无此功能。
} TIM_TimeBaseInitTypeDef;


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

## 实验：基本定时器定时

### 硬件设计

本实验利用基本定时器 TIM6/7 定时 1s，1s 时间到 LED 翻转一次。基本定时器是单片机内部的资源，没有外部 IO，不需要接外部电路，现只需要一个 LED 即可。

### 软件设计

> 1. 使能定时器时钟 TIMx_CLK, x[6,7]；  __TIMX_CLK_ENABLE()
> 2. 初始化时基初始化结构体；                HAL_TIM_Base_Init(TIM_MasterConfigTypeDef *htimx)
> 3. 使能 TIMx, x[6,7] update 中断；
> 4. 启动定时器；
> 5. 编写中断服务程序

#### 软件分析

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
    TIM_MasterConfigTypeDef sMasterConfig;	//配置定时器主模式（Master Mode）参数的结构体。它的核心作用是配置定时器如何通过 TRGO（触发输出）信号来同步或触发其他外设（如其他定时器、ADC、DAC 等）

    htimx.Instance = BASIC_TIMx;
    htimx.Init.Prescaler = BASIC_TIMx_PRESCALER;
    htimx.Init.CounterMode = TIM_COUNTERMODE_UP;
    htimx.Init.Period = BASIC_TIMx_PERIOD;
    HAL_TIM_Base_Init(&htimx);
    
     // 启动定时器中断
    if (HAL_TIM_Base_Start_IT(&htimx) != HAL_OK)
    {
        Error_Handler();
    }
}


// TIM6底层硬件初始化（HAL回调）
void HAL_TIM_Base_MspInit(TIM_HandleTypeDef *htim)
{
    if (htim->Instance == TIM6)
    {
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

> 在中断服务程序的最后，要把相应的中断标志位清除掉，切记。

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
    while(1)
    {
        if ( time == 1000 ) /* 1000 * 1 ms = 1s 时间到 */
        {
            time = 0;   
            LED1_TOGGLE; 	/* LED1 取反 */      
        }        
    }
}
```

