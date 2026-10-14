

# RTC 实时时钟

>[STM32 HAL库RTC实时时钟超细详解](https://blog.csdn.net/m0_75187370/article/details/147149171)
>
>功能
>
>1. 纯粹的计时
>2. 触发中断
>   - 秒中断，用来产生一个可编程的周期性中断信号；
>   - 闹钟中断，用来产生一个可编程的闹钟中断；
>   - 溢出中断，指示内部可编程计数器溢出并回转为0的状态。

## 一、Unix时间戳

- Unix 时间戳：为从UTC的1970年1月1日0时0分0秒开始所经过的秒数。
- UTC时间（协调世界时）：全球通用的最主要时间标准。
- 本地时间：世界上所有时区的秒计数器相同，不同时区通过添加偏移来得到当地时间。
- 时间戳存储在一个秒计数器中，秒计数器为32位/64位的整型变量。

<img src="C:/Users/acer/AppData/Roaming/Typora/typora-user-images/image-20260813183324882.png" alt="image-20260813183324882" style="zoom:60%;" />

## 二、RTC简介

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/d03ab7ca15173f6f1b33c481cfb2fd8c.png" alt="在这里插入图片描述" style="zoom: 33%;" />

> RTC是嵌入式系统中用于保持时间和日期的重要外设。即使在系统主电源断开的情况下，RTC也能通过备用电池继续工作，确保时间信息不丢失（RTC位于备份域中，当主电源 VDD 断电时，由VBAT引脚供电）。RTC广泛应用于需要时间戳、定时任务、日志记录等场景。
>
> STM32的RTC外设实质是一个掉电后还继续运行的定时器。它的复杂之处并不在于它的定时功能，而在于它掉电还继续运行的特性。

- RTC（Real Time Clock）实时时钟
- RTC是一个独立的定时器，可为系统提供时钟和日历功能，支持定时中断
- RTC和时钟配置系统处于后备区域，系统复位时数据不清零，VDD（2.0~3.6V）断电后可借助VBAT（1.8~3.6V）供电继续走时
- 32位的可编程计数器，可对应Unix时间戳的秒计数器
- 20位的可编程预分频器，可适配不同频率的输入时钟
- RTC 通常使用 **LSE（32.768 kHz 外部晶振）** 作为时钟源，精度高且功耗低。

**可选RTC时钟源：**

- LSE时钟（通常为32.768KHz）
- HSE的128分频（通常为8MHz/128）
-  LSI时钟（40KHz）

​		RTC使用的时钟源有三种，分别为高速外部时钟 的128分频（HSE/128）、低速内部时钟 LSI 、低速外部时钟 LSE ；使HSE分频时钟或LSI的话，在主电源VDD掉电的情况下，这两个时钟来源都会受到影响，没法保证RTC正常工作。因此RTC一般使用低速外部时钟LSE。

> **预分频器计算**（以 LSE = 32.768 kHz 为例）：
>
> - 异步预分频 `AsynchPrediv` = 127
> - 同步预分频 `SynchPrediv` = 255
>
> 最终 RTC 时钟频率 = 32768 / ((127+1) × (255+1)) = **1 Hz**，即每秒更新一次日历。

## 三、RTC架构

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image-20260813214641035.png" alt="image-20260813214641035" style="zoom: 67%;" />

>框图中浅灰色的部分属于备份域，在VDD掉电时可在VBAT的驱动下继续运行。这部分仅包括RTC的分频器，计数器和闹钟控制器。
>
>若VDD电源有效，RTC可触发秒中断、溢出事件和闹钟中断。 若STM32原本处于待机状态，可由闹钟事件或WKUP事件(外部唤醒事件，属于EXTI模块，不属于RTC) 使它退出待机模式。
>
>在配置RTC模块的时钟时，通常把输入的32.768KHz的 RTCCLK 进行32768分频得到实际驱动计数器的时钟TR_CLK = RTCCLK/32768= 1 Hz，计时周期为1秒，计时器在TR_CLK的驱动下计数，即每秒计数RTC_CNT的值加1。
>
>由于备份域的存在，使得RTC核具有了完全独立于APB1接口的特性，也因此对RTC寄存器的访问要遵守一定的规则。
>系统复位后，默认禁止访问后备寄存器和RTC，防止对后备区域(BKP)的意外写操作。执行以下操作使能对后备寄存器和RTC的访问：
>
>- 设置RCC_APB1ENR寄存器的PWREN和BKPEN位来使能电源和后备接口时钟。
>- 设置PWR_CR寄存器的DBP位使能对后备寄存器和RTC的访问。
>

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image-20260813215250353.png" alt="image-20260813215250353" style="zoom: 67%;" />

## 四、RTC相关结构体及函数

> HAL库函数对每个外设都建立了一个初始化结构体，比如RTC_InitTypeDef，结构体成员用于设置外设工作参数，并由外设初始化配置函数，比如RTC_Init()调用，这些配置好的参数将会设置外设相应的寄存器，达到配置外设工作环境的目的。

```C
//RTC初始化结构体
typedef struct
{
  uint32_t AsynchPrediv;  // RTC_CLK的异步分频因子
  uint32_t OutPut;        // RTC输出选择:无/秒输出/闹钟输出/校准输出
} RTC_InitTypeDef;
HAL_RTC_Init(RTC_HandleTypeDef *hrtc);

//RTC时间结构体
typedef struct {
	uint8_t Hours;		// 小时设置，0~11
	uint8_t Minutes;  	// 分钟设置，0~59
	uint8_t Seconds;  	// 秒设置，0~59
} RTC_TimeTypeDef;
HAL_StatusTypeDef HAL_RTC_SetTime(RTC_HandleTypeDef *hrtc, RTC_TimeTypeDef *sTime, uint32_t Format)

//RTC日期结构体
typedef struct {
    uint8_t WeekDay; 	// 星期几设置
    uint8_t Month;   	// 月份设置
    uint8_t Date;    	// 日期设置
    uint8_t Year;    	// 年份设置
} RTC_DateTypeDef;
HAL_StatusTypeDef HAL_RTC_SetDate(RTC_HandleTypeDef *hrtc, RTC_DateTypeDef *sDate, uint32_t Format)

//RTC闹钟结构体
typedef struct {
    RTC_TimeTypeDef AlarmTime;// 设定RTC时间寄存器的值：时/分/秒
    uint32_t Alarm;           // RTC闹钟选择，即选择闹钟A或者闹钟B。
} RTC_AlarmTypeDef;  
//设置 RTC 闹钟并启用中断的函数
HAL_RTC_SetAlarm_IT(RTC_HandleTypeDef *hrtc, RTC_AlarmTypeDef *sAlarm, uint32_t Format);
```

## 五、实验：利用RTC提供北京时间

RTC外设是个连续计数的计数器，利用它提供的时间戳，可通过程序转换输出实时时钟和日历的功能， 修改计数器的值则可以重新设置系统当前的时间和日期。由于它的时钟配置系统（RCC_BDCR 寄存器）是在备份域， 在系统复位或从待机模式唤醒后RTC的设置维持不变，而且使用备份域电源可以全RTC计时器在主电源关掉的情况下仍然运行，保证时间的正确。

### 硬件设计

开发板中与RTC相关的硬件设计见图。

<img src="https://doc.embedfire.com/mcu/stm32/f103/hal_general/zh/latest/_images/image410.jpeg" alt="图 39‑3 RTC硬件电路" style="zoom: 50%;" />

> 原理图左上角是备份域的供电电路，在本开发板中提供了一个钮扣电池插槽，可以接入型号为CR1220的钮扣电池，电池电压为3.2V，图中的BAT54C双向二极管可切换输入到STM32备份域电源引脚VBAT的供电，当主电源正常供电时，由稳压器输出的3.3V供电，当主电源掉电时，由钮扣电池供电。
>
> 原理图下方是本开发板采用的LSE晶振电路，此处使用的晶振频率为32.768KHz，RTC外设可以使用LSE作为时钟，把它进行分频得到1Hz的RTC计时时钟。

注意：本实验默认使用LSI内部时钟，使用内部时钟时，即使安装了钮扣电池，主电源掉电后时间是不会继续走的，只会保留上次断电的时间。若要持续运行，需要修改bsp_rtc.h文件，使用RTC_CLOCK_SOURCE_LSE宏，切换成使用LSE外部时钟。

### 软件设计                                                

> 1. 初始化RTC外设与时钟设置； 
> 2. 设置时间以及添加配置标志；
> 3. 获取当前时间；
