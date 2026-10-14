# STM32的RCC外设

RCC（Reset and Clock Control，复位与时钟控制），负责管理STM32芯片的复位和时钟的硬件单元。  

1. 复位：STM32的复位有系统复位、电源复位和后备域复位三种复位。

   - 系统复位。复位除时钟控制寄存器（CSR）中的复位标志和备份域中的寄存器外的所有寄存器。

   - 电源复位。复位除备份域外的所有寄存器。

   - 后备域复位。只复位备份域。

2. 时钟：由电路产生的具有周期性的脉冲信号，相当于单片机的心脏，其跳动频率（时钟速度）直接决定了芯片的性能、功耗以及各外设能否正常工作。时钟来源如下
   - 三种不同的时钟源可被用来驱动系统时钟(SYSLCLK)：
     - HSI、HSE、PLL。
   - 二级时钟源：
     - 40kHz LSI；
     - 32.768KHz LSE；

> STM32时钟系统主要的目的就是给相对独立的外设模块提供时钟，主要也是为了降低整个芯片的功耗，所有外设时钟默认都是关闭状态(disable)，当使用某个外设就要开启该外设的时钟(enable)，不同外设需要的时钟频率不同，没必要所有外设都用高速时钟造成浪费，而且有些外设也接受不了这么高的频率，这也是为什么STM32有四个时钟源的原因，就是兼容不同速度的外设，STM32的四个时钟源分别为：HSE、 LSE、HSI、LSI。
>STM32用多个时钟源、像树一样分叉的时钟树，来给CPU和各个外设提供不同频率的时钟信号。

## 复位

STM32F10xxx支持三种复位形式，分别为系统复位、上电复位和备份区域复位。

1. 系统复位

   除了时钟控制器的RCC_CSR寄存器中的复位标志位和备份区域中的寄存器以外，系统复位将复位所有寄存器至它们的复位状态。

    当发生以下任一事件时，产生一个系统复位： 

   - NRST引脚上的低电平（外部复位） 
   - 窗口看门狗计数终止（WWDG复位） 
   - 独立看门狗计数终止（IWDG复位） 
   - 软件复位（SW复位） 
   - 低功耗管理复位 

   可通过查看RCC_CSR控制状态寄存器中的复位状态标志位识别复位事件来源。

2. 电源复位

   电源复位将复位除了备份区域外的所有寄存器。

   当以下事件中之一发生时，产生电源复位： 

   - 上电/掉电复位(POR/PDR复位) 
   - 从待机模式中返回 

3. 备份域复位

   备份区域拥有两个专门的复位，它们只影响备份区域。 

   当以下事件中之一发生时，产生备份区域复位：

   - 软件复位，备份区域复位可由设置备份域控制寄存器 (RCC_BDCR)中的BDRST位产生。
   - 在VDD和VBAT两者掉电的前提下，VDD或VBAT上电将引发备份区域复位。

## 时钟控制

三种不同的时钟源可被用来驱动系统时钟(SYSCLK)：

- HSI振荡器时钟 
- HSE振荡器时钟
-  PLL时钟 

这些设备有以下2种二级时钟源：

- 40kHz 低速LSI，可以用于驱动独立看门狗和通过程序选择驱动RTC。RTC用于从停机/待机模式下自动唤醒系统。 
- 32.768kHz低速LSE，可用来通过程序选择驱动RTC(RTCCLK)。 当不被使用时，任一个时钟源都可被独立地启动或关闭，由此优化系统功耗。

![image-20260830110857760](https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image-20260830110857760.png)

**五大时钟源**

STM32通常有5个时钟源，可以满足从高性能到低功耗的各种需求。

| 时钟源  | 全称         | 类型                 | 典型频率                          | 主要用途                                                     |
| :------ | :----------- | :------------------- | :-------------------------------- | :----------------------------------------------------------- |
| **HSI** | 高速内部时钟 | 内部RC振荡器         | 8MHz (F1系列) / 16MHz (F4系列)    | 默认时钟源，精度一般，启动快                                 |
| **LSI** | 低速内部时钟 | 内部RC振荡器         | 约32-40kHz                        | 功耗极低，用于独立看门狗(IWDG)和RTC的备用时钟源（低功耗模式）。 |
| **HSE** | 高速外部时钟 | 外部晶振/时钟源      | 4-26MHz (常用8MHz)                | 精度高，常作为**主系统时钟源**                               |
| **LSE** | 低速外部时钟 | 外部晶振 (32.768kHz) | 32.768kHz                         | 精度高，**实时时钟(RTC)**                                    |
| **PLL** | 锁相环       | 倍频器（非独立源）   | 最高可达72MHz (F1) 或 168MHz (F4) | 将（来自HSI或HSE的）较低频率倍频到高频，供系统使用。         |

> 上电默认：芯片先用 HSI 启动运行，用户代码里再切换到 HSE\+PLL 高频时钟。
>

### 时钟树层级流向

STM32的时钟系统通常用“时钟树”来表示，形象地描述了时钟信号从根源到各个分支的流动路径。

- 整个时钟分为三层：时钟源 → 系统主时钟树 \(SYSCLK/AHB/APB\) → 各个外设分支时钟。
- "树"的主干是**系统时钟(SYSCLK)**。它的主要来源是：HSI、HSE或PLL。通常会将HSE或HSI接入PLL进行倍频，然后将PLL输出作为SYSCLK，以获得更高的性能。
- 从SYSCLK这个主干出发，时钟信号会经过一系列"分支"，为不同总线和外设提供时钟。

```
系统时钟(SYSCLK)来源：
	HSE(8M)/HSI → PLL倍频因子 → SYSCLK(系统主时钟)

系统主时钟树：
    SYSCLK
      → AHB预分频 → AHB总线时钟HCLK（为内核、内存和DMA等高速组件提供时钟）
      → APB1预分频 → APB1总线时钟PCLK1（为低速外设提供时钟：I2C、USART2、定时器TIM2~5等）
      → APB2预分频 → APB2总线时钟PCLK2（为高速外设提供时钟：GPIO、USART1、SPI1、定时器TIM1、EXTI等）
```

### RCC时钟框图刨析

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image24.png" alt="STM32时钟树" style="zoom: 80%;" />

**系统时钟**

①HSE高速外部时钟信号

> HSE是高速的外部时钟信号，可以由有源或者无源晶振提供，频率从4-16MHZ不等。当使用有源晶振时，时钟从OSC_IN引脚进入，OSC_OUT引脚悬空，当选用无源晶振时，时钟从OSC_IN和OSC_OUT进入，并且要配谐振电容。
>
> HSE最常使用的就是8M的无源晶振。当确定PLL时钟来源的时候，HSE可以不分频或者2分频，这个由时钟配置寄存器CFGR的位17：PLLXTPRE设置，我们设置为HSE不分频。

②PLL时钟源

> PLL时钟来源可以有两个：HSE或HSI/2，具体用哪个由时钟配置寄存器CFGR的位16：PLLSRC设置。HSI是内部高速的时钟信号，频率为8M，根据温度和环境的情况频率会有漂移，一般不作为PLL的时钟来源。这里我们选HSE作为PLL的时钟来源。

 ③PLL时钟PLLCLK

> 通过设置PLL的倍频因子，可以对PLL的时钟来源进行倍频，倍频因子可以是：[2,3,4,5,6,7,8,9,10,11,12,13,14,15,16]，具体设置成多少，由时钟配置寄存器CFGR的位21-18：PLLMUL[3:0]设置。我们这里设置为9倍频，因为上一步我们设置PLL的时钟来源为HSE=8M，所以经过PLL倍频之后的PLL时钟：PLLCLK = 8M *9 = 72M。
>
> 72M是ST官方推荐的稳定运行时钟，如果你想超频的话，增大倍频因子即可，最高为128M。 我们这里设置PLL时钟：PLLCLK = 8M *9 = 72M。

 ④系统时钟SYSCLK

> 系统时钟来源可以是：HSI、PLLCLK、HSE，具体的时钟配置寄存器CFGR的位1-0：SW[1:0]设置。我们这里设置系统时钟：SYSCLK = PLLCLK = 72M。

⑤AHB总线时钟HCLK

> APB总线时钟HCLK由SYSCLK经过AHB预分频器分频后得到，分频因子可以是:[1,2,4,8,16,64,128,256,512]，具体的由时钟配置寄存器CFGR的位7-4 ：HPRE[3:0]设置。
>
> 片上大部分外设的时钟都是经过HCLK分频得到，至于AHB总线上的外设的时钟设置为多少，得等到我们使用该外设的时候才设置，我们这里只需粗线条的设置好APB的时钟即可。我们这里设置为1分频，即HCLK=SYSCLK=72M。

⑥APB2总线时钟PCLK2

> APB2总线时钟PCLK2由HCLK经过高速APB2预分频器得到，分频因子可以是:[1,2,4,8,16]，具体由时钟配置寄存器CFGR的位13-11：PPRE2[2:0]决定。APB2属于高速的总线时钟，片上高速的外设就挂载到这条总线上。
>
> 至于APB2总线上的外设的时钟设置为多少，得等到我们使用该外设的时候才设置，我们这里只需粗线条的设置好APB2的时钟即可。我们这里设置为1分频，即PCLK2 = HCLK = 72M。

⑦APB1总线时钟PCLK1

> APB1总线时钟PCLK1由HCLK经过低速APB预分频器得到，分频因子可以是:[1,2,4,8,16]，具体由时钟配置寄存器CFGR的位10-8：PRRE1[2:0]决定。 PCLK1属于低速的总线时钟，最高为36M，片上低速的外设就挂载到这条总线上。
>
> 至于APB1总线上的外设的时钟设置为多少，得等到我们使用该外设的时候才设置，我们这里只需粗线条的设置好APB1的时钟即可。 我们这里设置为2分频，即PCLK1 = HCLK/2 = 36M。

**其他时钟**

A、USB时钟

> USB时钟是由PLLCLK经过USB预分频器得到，分频因子可以是：[1,1.5]，具体的由时钟配置寄存器CFGR的位22：USBPRE配置。USB的时钟最高是48M，根据分频因子反推过来算，PLLCLK只能是48M或者是72M。一般我们设置PLLCLK=72M，USBCLK=48M。USB对时钟要求比较高，所以PLLCLK只能是由HSE倍频得到，不能使用HSI倍频。

B、Cortex系统时钟

> Cortex系统时钟由HCLK 8分频得到，等于9M，Cortex系统时钟用来驱动内核的系统定时器SysTick，SysTick一般用于操作系统的时钟节拍，也可以用做普通的定时。

C、ADC时钟

> ADC时钟由PCLK2经过ADC预分频器得到，分频因子可以是[2,4,6,8]，具体的由时钟配置寄存器CFGR的位15-14：ADCPRE[1:0]决定。
>
> 很奇怪的是怎么没有1分频。ADC时钟最高只能是14M，如果采样周期设置成最短的1.5个周期的话，ADC的转换时间可以达到最短的1us。如果真要达到最短的转换时间1us的话，那ADC的时钟就得是14M，反推PCLK2的时钟只能是：28M、56M、84M、112M，鉴于PCLK2最高是72M，所以只能取28M和56M。

D、RTC时钟、独立看门狗时钟

> RTC时钟可由HSE/128分频得到，也可由低速外部时钟信号LSE提供，频率为32.768KHZ，也可由低速内部时钟信号LSI提供，具体选用哪个时钟由备份域控制寄存器BDCR的位9-8：RTCSEL[1:0]配置。
>
> 独立看门狗的时钟由LSI提供，且只能是由LSI提供，LSI是低速的内部时钟信号，频率为30~60KHZ直接不等，一般取40KHZ。

### 时钟配置

> 设置系统时钟SYSCLK、AHB分频因子（决定HCLK值）、APB2分频因子（决定PCLK2值）、APB1分频因子（决定PCLK1值）、各个外设的分频因子；控制AHB、APB2和APB1这三条总线时钟的开启、控制每个外设的时钟的开启。
>
> 对于SYSCLK、HCLK、PCLK2、PCLK1这四个时钟的配置一般是：PCLK2 = HCLK = SYSCLK=PLLCLK = 72M，PCLK1=HCLK/2 = 36M。

1. 时钟配置方式（3种）

   - STM32CubeMX图形化配置（推荐）：在`Clock Configuration`界面可视化配置，自动生成代码，方便直观且不易出错。

   - HAL库函数：通过调用`HAL_RCC_OscConfig`等标准函数配置。

   - 直接操作寄存器：直接操作RCC（复位与时钟控制）寄存器，效率最高但最复杂。

2. 时钟配置相关寄存器

   | 寄存器                    | 主要功能                                                     |
   | :------------------------ | :----------------------------------------------------------- |
   | `RCC_CR `时钟控制寄存器   | 时钟源控制。使能/失能HSI、HSE、PLL等时钟源，并读取其就绪标志。 |
   | `RCC_CFGR` 时钟配置寄存器 | 时钟路径分配。配置系统时钟(SYSCLK)来源、AHB/APB预分频器、PLL倍频系数等核心参数。负责配置时钟的分频、倍频、路由和输出等。 |

## 时钟系统时钟配置函数

STM32上电执行的第一个程序：STM32单片机上电，就会执行启动文件（汇编代码编写）中的复位程序：

1. 调用SystemInit() 系统初始化函数完成系统时钟的配置(配置成72MHZ)
2. 调用_main函数初始化堆栈指针，然后再调用C库函数main函数，去到C语言的世界
   所以跳转到C语言的main函数时，已经完成了系统时钟(SYSCLK)的配置。

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image-20260811153521493.png" alt="image-20260811153521493" style="zoom: 33%;" />

## 实验：配置系统时钟

一般情况下都是使用HSE，然后HSE经过PLL倍频之后作为系统时钟。通常的配置是：HSE=8MHz，PLL的倍频因子为9，系统时钟就设置成：SYSCLK = 8M * 9 = 72MHz。使用HSE，系统时钟最高为72M，这个是官方推荐的最高的稳定时钟。

### 软件设计

1. 使用HSE配置系统时钟

```c
/**
  * @brief  系统时钟配置函数
  * @note   本函数假设外部高速时钟（HSE）为 8MHz，通过 PLL 倍频至 72MHz，
  *         最终 SYSCLK = 72MHz, HCLK = 72MHz, PCLK2 = 72MHz, PCLK1 = 36MHz。
  * @retval 无
  */
void SystemClock_Config(void)
{
    RCC_ClkInitTypeDef clkinitstruct = {0};	//定义时钟初始化结构体变量（用于配置系统时钟源及分频）
    RCC_OscInitTypeDef oscinitstruct = {0};	//定义振荡器初始化结构体变量（用于配置HSE、PLL等振荡器）

    /*-------------------- 步骤1：配置振荡器（HSE 和 PLL） --------------------*/
    oscinitstruct.OscillatorType  = RCC_OSCILLATORTYPE_HSE;	//配置振荡器类型为HSE
    oscinitstruct.HSEState        = RCC_HSE_ON;	//开启HSE外部晶振
    oscinitstruct.HSEPredivValue  = RCC_HSE_PREDIV_DIV1; //HSE预分频器：不分频（即HSE频率=8MHz）
    oscinitstruct.PLL.PLLState    = RCC_PLL_ON; //配置PLL使能
    oscinitstruct.PLL.PLLSource   = RCC_PLLSOURCE_HSE; //PLL时钟源选择HSE
    oscinitstruct.PLL.PLLMUL      = RCC_PLL_MUL9; //PLL倍频系数：9倍频，即8×9=72MHz

  // 调用HAL库函数配置上述振荡器
  if (HAL_RCC_OscConfig(&oscinitstruct) != HAL_OK)
  {
    while(1); 	// 初始化错误处理：程序停止
  }

  /*-------------------- 步骤2：配置系统时钟及总线分频 --------------------*/
  // 选择需要配置的时钟类型：系统时钟（SYSCLK）、AHB、APB1、APB2
  clkinitstruct.ClockType = (RCC_CLOCKTYPE_SYSCLK | RCC_CLOCKTYPE_HCLK | 
                             RCC_CLOCKTYPE_PCLK1 | RCC_CLOCKTYPE_PCLK2);
  clkinitstruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;	//选择PLL输出作为系统时钟源（SYSCLK = 72MHz）
  clkinitstruct.AHBCLKDivider = RCC_SYSCLK_DIV1; //AHB总线时钟（HCLK）= SYSCLK / 1 = 72MHz	
  clkinitstruct.APB2CLKDivider = RCC_HCLK_DIV1;	//APB2高速总线时钟（PCLK2）= HCLK / 1 = 72MHz
  clkinitstruct.APB1CLKDivider = RCC_HCLK_DIV2; //APB1 低速总线时钟（PCLK1）= HCLK / 2 = 36MHz

  // 调用HAL库函数配置系统时钟，并设置Flash等待周期为2个周期（匹配 72MHz工作频率）
  if (HAL_RCC_ClockConfig(&clkinitstruct, FLASH_LATENCY_2) != HAL_OK)
  {
    while(1); 	// 初始化错误处理：程序停止
  }
}
```

2. 主函数

> 在主函数中，可以调用HSE_SetSysClock()或者HSI_SetSysClock()这两个函数把系统时钟设置成各种常用的时钟，然后通过MCO引脚监控，或者通过LED闪烁的快慢体验不同的系统时钟对同一个软件延时函数的影响。

```c
int main(void)
{
     SystemClock_Config();	// 使用HSE，配置系统时钟为72M
    LED_GPIO_Config();		// LED 端口初始化
    while (1)
    {
        LED2( ON );      // 亮
        Delay(0x0FFFFF);
        LED2( OFF );     // 灭
        Delay(0x0FFFFF);
    }
}
```