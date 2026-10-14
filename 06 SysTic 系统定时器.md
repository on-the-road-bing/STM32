# SysTick 系统定时器

## 一、SysTick简介

> 前言：为什么SysTick是嵌入式开发的"瑞士军刀"？
> 在STM32开发中，经常需要精确的延时功能或周期性任务调度。实现这些功能的方式有很多，比如使用外设定时器（TIM2-TIM5），但这类定时器往往需要占用GPIO引脚和外设资源。
>
> 而Cortex-M内核自带的SysTick（系统定时器，即系统嘀嗒定时器） 完美解决了这一问题——SysTick属于CM3内核中的一个外设，内嵌在NVIC中，无需占用外设资源，可直接用于系统延时、RTOS任务调度等核心功能。。系统定时器是一个24bit的递减计数器，最大计数值为全F。计数器每计数一次的时间为1/SYSCLK，当计数减到0时，系统定时器就产生一次中断，并从 RELOAD 寄存器中自动重装载定时初值，开始新一轮计数，以此循环往复。
>
> Systick 的信号来源于系统时钟SYSCLK，可以选择不分频或8分频，分别对应72MHz和9MHz的频率。
>
> 系统定时器一般用于操作系统，用于产生时基，维持操作系统的心跳。

**SysTick的核心特性**

| 特性     | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| 位数     | 24位递减计数器，最大计数值为2^24-                            |
| 计数模式 | 递减计数（从装载值减到0后自动重装载）                        |
| 时钟源   | 可选两种时钟源：SYSCLK、SYSCLK/8                             |
| 中断支持 | 计数到0时可触发中断（SysTick_IRQn）                          |
| 资源占用 | 内核集成，不占用外设定时器资源                               |
| 典型应用 | 系统延时函数（delay_ms/delay_us）、RTOS任务调度、周期性任务触发 |

**SysTick寄存器汇总**

| 寄存器名称 | 寄存器描述              | 作用                           |
| ---------- | ----------------------- | ------------------------------ |
| CTRL       | SysTick控制及状态寄存器 | 用于控制定时器的启停和状态监控 |
| LOAD       | SysTick重装载数值寄存器 | 设置定时器的重装载值           |
| VAL        | SysTick当前数值寄存器   | 显示当前的倒计数值             |
| CALIB      | SysTick校准数值寄存器   | 提供校准值，用于精确时间基准   |

## 二、SysTick配置库函数

> 用固件库编程的时候只需要调用库函数SysTick_Config()即可，形参ticks用来设置重装载寄存器的值， 最大不能超过重装载寄存器的值2^24，当重装载寄存器的值递减到0的时候产生中断， 然后重装载寄存器的值又重新装载往下递减计数，以此循环往复。紧随其后设置中断优先级， 最后配置系统定时器的时钟等于AHBCLK=72M，使能定时器和定时器中断，这样系统定时器就配置好了，一个库函数搞定。
>
> 在HAL库中，`HAL_Init()` 函数会调用 `HAL_SYSTICK_Config()` 来完成SysTick的初始化。它最终也会调用 `SysTick_Config()`。HAL库的 `HAL_Delay()` 延时函数就是基于SysTick实现的。

```C
 __STATIC_INLINE uint32_t SysTick_Config(uint32_t ticks)
 {
     // 不可能的重装载值，超出范围
     if ((ticks - 1UL) > SysTick_LOAD_RELOAD_Msk)
         return (1UL);
     SysTick->LOAD  = (uint32_t)(ticks - 1UL);	// 设置重装载寄存器
     NVIC_SetPriority (SysTick_IRQn, (1UL << __NVIC_PRIO_BITS) - 1UL); // 设置中断优先级
     SysTick->VAL   = 0UL; 					  // 设置当前数值寄存器

     // 设置系统定时器的时钟源为AHBCLK=72M，使能系统定时器中断，使能定时器
     SysTick->CTRL  = SysTick_CTRL_CLKSOURCE_Msk |
                     SysTick_CTRL_TICKINT_Msk   |
                     SysTick_CTRL_ENABLE_Msk;
     return (0UL);
 }
```

## 三、SysTick定时实验（用户实现）

> 利用SysTick产生1s的时基，LED以1s的频率闪烁。

### 硬件设计

SysTick属于单片机内部的外设，不需要额外的硬件电路，剩下的只需一个LED灯即可。

### 软件设计

#### 代码分析

1. SysTick初始化函数

> SysTick初始化函数由用户编写，里面调用了HAL_SYSTICK_Config()这个固件库函数，通过设置该固件库函数的形参来决定系统定时器经过多少时间就产生一次中断。

```c
 void SysTick_Init(void)
 {
     /**
     * SysTick初始化函数：启动系统滴答定时器 SysTick
     * SystemFrequency / 1000    1ms中断一次
     * SystemFrequency / 100000  10us中断一次
     * SystemFrequency / 1000000 1us中断一次
     */
     if (HAL_SYSTICK_Config(SystemCoreClock / 100000)) {
         /* Capture error */
         while (1);
     }
 }
uint32_t HAL_SYSTICK_Config(uint32_t TicksNumb)
{
   return SysTick_Config(TicksNumb);
}
```

**SysTick中断时间计算**
SysTick定时器的计数器是向下递减计数的，计数一次的时间1/CLKAHB， 当重装载寄存器中的值减到0时产生中断。
HAL_SYSTICK_Config()形参的值最终写到重装载寄存器LOAD， 从而可知SysTick定时器中断一次的时间。

**SysTick定时时间计算**
当设置好中断时间后，通过设置一个变量t， 用来记录进入中断的次数，那么变量t乘以中断的时间就可以计算出需要定时的时间。

2. SysTick定时函数

> 函数Delay_us()中当TimingDelay为0的时候表示延时时间到。TimingDelay在中断函数中递减，即SysTick每进一次中断即10us的时间，TimingDelay递减一次。

```c
/**
* @brief   us延时程序,10us为一个单位
* @param
*   @arg nTime: Delay_us(1) 则实现的延时为 1 * 10us = 10us
*/
void Delay_us(__IO u32 nTime)
{
    TimingDelay = nTime;
    while (TimingDelay != 0);
}
```

3. SysTick中断服务函数（用户实现）

```c
void SysTick_Handler(void)
{
    TimingDelay_Decrement(); 
}

// 节拍递减程序
void TimingDelay_Decrement(void)
{
    if (TimingDelay != 0x00)
        TimingDelay--;
}
```

4. 主函数

> 主函数中初始化了LED和SysTick，然后在一个while循环中以1s的频率让LED闪烁。

```C
int main(void)
{
    SystemClock_Config();	/* 系统时钟初始化成72MHz */
    LED_GPIO_Config(); 		/* LED 端口初始化 */
    
    /* 
    * 配置SysTick为10us中断一次,时间到后触发定时中断，
    * 进入SysTick_Handler处理，通过数中断次数计时
    */
    SysTick_Init();
    while (1) {
        LED_RED;
        Delay_us(100000);     // 10000 * 10us = 1000ms
        LED_GREEN;
        Delay_us(100000);   // 10000 * 10us = 1000ms
        LED_BLUE;
        Delay_us(100000);   // 10000 * 10us = 1000ms
    }
}
```