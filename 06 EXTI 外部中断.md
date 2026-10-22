# EXTI 外部中断

## 一、中断系统

- 中断：执行程序的过程中，由于外部或内部事件的发生，使得CPU暂停当前正在运行的程序，转而去处理中断程序，处理完成后又返回原来被暂停的位置继续运行。
- 中断优先级：当多个中断源同时申请中断时，CPU会根据中断源的轻重缓急进行裁决，优先响应更加紧急的中断源。
- 中断嵌套：当一个中断程序正在运行时，又有新的更高优先级的中断源申请中断，CPU再次暂停当前中断程序，转而去处理新的中断程序，处理完成后依次进行返回。

中断执行流程示意图：

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/0ebd29a85e014784becd4c95c3aa1293.png" alt="0ebd29a85e014784becd4c95c3aa1293.png" style="zoom: 33%;" />

## 二、STM32中断

- 支持68个可屏蔽中断通道，其中系统异常有8个， 外部中断有60个。包含EXTI、TIM、ADC、USART、SPI、I2C、RTC等多个外设。
- 使用 NVIC 统一管理中断，每个中断通道都拥有16个可编程的优先等级，可对优先级进行分组，进一步设置抢占优先级和响应优先级。

### NVIC 简介

NVIC（嵌套向量中断控制器 ）是 STM32 中的中断控制器，负责处理中断请求，决定哪些中断需要立即响应，哪些可以稍后处理。它通过一系列的寄存器来控制中断的使能、优先级等。

工作原理：当一个中断事件发生时，NVIC会根据中断优先级和当前CPU状态来决定是否立即响应该中断。如果响应，CPU会跳转到相应的中断服务程序执行。

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image-20260811162340915.png" alt="image-20260811162340915" style="zoom:25%;" />

#### NVIC 相关结构体、函数

- 中断源
- NVIC 寄存器
- NVIC 中断配置固件库函数（基本不用，常用HAL NVIC配置函数）
- HAL库NVIC配置函数

```c
/********IRQn_Type中断源类型枚举********/
typedef enum {
    //Cortex-M3 处理器异常编号
    NonMaskableInt_IRQn   = -14,
    ......
    SysTick_IRQn          = -1, 
    
    //STM32 外部中断编号
    WWDG_IRQn             = 0,
    RTC_IRQn              = 3,
    RCC_IRQn              = 5,
    EXTI0_IRQn            = 6,	// EXTI线0中断
    EXTI1_IRQn            = 7,	// EXTI线1中断
    DMA1_Channel1_IRQn    = 11,
    DMA1_Channel2_IRQn    = 12, 
    EXTI9_5_IRQn          = 23, 
    EXTI15_10_IRQn        = 40, 
    ......
} IRQn_Type;

/**
 * NVIC寄存器简介：
 * 在固件库中，NVIC的结构体定义可谓是颇有远虑，给每个寄存器都预留了很多位，
 * 在配置中断的时候一般只用ISER、ICER和IP这三个寄存器，
 * ISER用来使能中断，ICER用来失能中断，IP用来设置中断优先级。
 */
/**********NVIC结构体定义**********/
typedef struct {
    __IOM uint32_t ISER[8U];            // 中断使能寄存器
    uint32_t RESERVED0[24U];
    __IOM uint32_t ICER[8U];            // 中断清除寄存器
    uint32_t RSERVED1[24U];
    __IOM uint32_t ISPR[8U];            // 中断使能悬起寄存器
    uint32_t RESERVED2[24U];
    __IOM uint32_t ICPR[8U];            // 中断清除悬起寄存器
    uint32_t RESERVED3[24U];
    __IOM uint32_t IABR[8U];            // 中断有效位寄存器/
    uint32_t RESERVED4[56U];
    __IOM uint8_t  IP[240U];            // 中断优先级寄存器(8Bit wide)
    uint32_t RESERVED5[644U];
    __OM  uint32_t STIR;                // 软件触发中断寄存器
}  NVIC_Type;

/**
 * NVIC 中断配置固件库函数：
 * 在固件库中，NVIC的结构体定义可谓是颇有远虑，给每个寄存器都预留了很多位，
 * 在配置中断的时候一般只用ISER、ICER和IP这三个寄存器，
 * ISER用来使能中断，ICER用来失能中断，IP用来设置中断优先级。
 */
/**********符合CMSIS标准的NVIC库函数**********/
void NVIC_EnableIRQ(IRQn_Type IRQn)						//使能中断
void NVIC_DisableIRQ(IRQn_Type IRQn)					//失能中断
void NVIC_SetPendingIRQ(IRQn_Type IRQn)					//设置中断悬起位
void NVIC_ClearPendingIRQ(IRQn_Type IRQn)				//清除中断悬起位
uint32_t NVIC_GetPendingIRQ(IRQn_Type IRQn)				//获取悬起中断编号
void NVIC_SetPriority(IRQn_Type IRQn, uint32_t priority) //设置中断优先级
uint32_t NVIC_GetPriority(IRQn_Type IRQn)			    //获取中断优先级
void NVIC_SystemReset(void)							  //系统复位
```

**HAL库 NVIC 中断配置函数**

```c
/**********HAL库NVIC函数**********/
HAL_NVIC_SetPriorityGrouping(IRQn_Type IRQn, uint32_t PreemptPriority, uint32_t SubPriority);		//HAL库中断优先级分组设置
HAL_NVIC_SetPriority(IRQn_Type IRQn, uint32_t PreemptPriority, uint32_t SubPriority)  //设置中断线的优先级
HAL_NVIC_EnableIRQ(IRQn_Type IRQn)	//使能指定的中断
```

### 中断优先级

NVIC 中断优先级寄存器 NVIC_IPRx 用来配置外部中断的优先级。NVIC_IPRx宽度为8bit，但只使用了高4bit。

<img src="图片和附件/FnHAh8O8.png" alt="FnHAh8O8\.png" style="zoom:50%;" />

用于表达优先级的4bit，又被分组成抢占优先级和子优先级。
		数值越小优先级越高。如果有多个中断同时响应，抢占优先级高的就会抢占抢占优先级低的优先得到执行， 如果抢占优先级相同，就比较子优先级。如果抢占优先级和子优先级都相同的话，就比较他们的硬件中断编号，编号越小，优先级越高。

优先级的分组由内核外设SCB的应用程序中断及复位控制寄存器AIRCR的PRIGROUP[10:8]位决定，F103分为了5组。

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/sDI3qBCl.png" alt="sDI3qBCl\.png" style="zoom: 50%;" />

```c
HAL_NVIC_SetPriorityGrouping(IRQn_Type IRQn, uint32_t PreemptPriority, uint32_t SubPriority);		//HAL库中断优先级分组设置
```

### 中断编程

1. 使能外设某个中断，具体由每个外设的相关中断使能位控制。
2. 配置外设中断源、中断优先级。
3. 编写中断服务函数

#### 常用外设中断函数概览

STM32的常见外设中断非常丰富，从简单的引脚触发到复杂的通信和定时器控制都有涉及。

| 外设类型        | 中断向量入口函数 (stm32f1xx_it.c中)       | HAL库通用处理函数                               | 用户实现回调函数                                             | 典型应用                      |
| :-------------- | :---------------------------------------- | :---------------------------------------------- | :----------------------------------------------------------- | ----------------------------- |
| 外部中断 (EXTI) | `EXTIx_IRQHandler`                        | `HAL_GPIO_EXTI_IRQHandler`                      | `HAL_GPIO_EXTI_Callback`                                     | 按键检测、外部信号触发        |
| DMA             | `DMAx_Streamx_IRQHandler`                 | `HAL_DMA_IRQHandler`                            | `HAL_DMA_XferCpltCallback`                                   | 高速批量数据传输 (内存到外设) |
| USART           | `USARTx_IRQHandler`                       | `HAL_UART_IRQHandler`                           | `HAL_UART_TxCpltCallback` `HAL_UART_RxCpltCallback`          | 数据收发、串口通信            |
| I2C             | `I2C1_EV_IRQHandler` `I2C1_ER_IRQHandler` | `HAL_I2C_EV_IRQHandler` `HAL_I2C_ER_IRQHandler` | `HAL_I2C_MasterTxCpltCallback` `HAL_I2C_SlaveRxCpltCallback`等 | 传感器、EEPROM读写            |
| SPI             | `SPIx_IRQHandler`                         | `HAL_SPI_IRQHandler`                            | `HAL_SPI_TxRxCpltCallback`                                   | 显示屏、闪存、ADC通信         |
| 定时器(TIM)     | `TIMx_IRQHandler`                         | `HAL_TIM_IRQHandler`                            | `HAL_TIM_PeriodElapsedCallback`                              | 延时、PWM、输入捕获、编码器   |
| ADC             | `ADCx_IRQHandler`                         | `HAL_ADC_IRQHandler`                            | `HAL_ADC_ConvCpltCallback`                                   | 模拟信号采集                  |
| 实时时钟 (RTC)  | `RTC_IRQHandler`                          | (由HAL库处理)                                   | `HAL_RTC_AlarmAEventCallback` 等                             | 闹钟唤醒、定时事件            |
| 电源/电压检测   | `PVD_IRQHandler`                          | (由HAL库处理)                                   | `HAL_PVD_PVDCallback`                                        | 低电压报警、电源管理          |
| 窗口看门狗      | `WWDG_IRQHandler`                         | (由HAL库处理)                                   | `HAL_WWDG_EarlyWakeupCallback`                               |                               |

#### EXTI 外部中断编程

```c
/********* 选择中断源，设置线路优先级 ********/
HAL_NVIC_SetPriority(IRQn_Type IRQn, uint32_t PreemptPriority, uint32_t SubPriority);	//选择中断源，设置中断线优先级
HAL_NVIC_EnableIRQ(IRQn_Type IRQn);	//使能指定的中断
/*******************************************/

/************** EXTI 外部中断 **************/
// 中断入口（启动文件定义，自行实现）
void EXTI0_IRQHandler(void);
...
void EXTI15_10_IRQHandler(void);

// HAL库标准中断分发函数
void HAL_GPIO_EXTI_IRQHandler(uint16_t GPIO_Pin);

// 中断回调（用户实现）
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin);
/*******************************************/

/************** EXTI外部中断示例模板 **************/
void EXTI15_10_IRQHandler(void)
{
  HAL_GPIO_EXTI_IRQHandler(GPIO_Pin);
}

void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
  if(GPIO_Pin == GPIO_PIN_13)
  {
    // 按键业务
  }
}
/***********************************************/
```

## 三、EXTI外部中断

EXTI（External interrupt/event controller ）——外部中断/事件控制器，它管理了控制器的20个中断/事件线。

- EXTI可以监测指定GPIO口的电平信号，当其指定的GPIO口产生电平变化时，EXTI将立即向NVIC发出中断申请，经过NVIC裁决后即可中断CPU主程序，使CPU执行EXTI对应的中断程序；
- 支持的触发方式：上升沿/下降沿/双边沿/软件触发；
- 支持的GPIO口：所有GPIO口，但相同的Pin不能同时触发中断，每条中断线只能对应一个触发源（PA1和PB1不能同时使用）；
- 通道数：16个GPIO_Pin，外加PVD输出、RTC闹钟、USB唤醒、以太网唤醒，共20个中断/事件线；
- 触发响应方式：中断响应（申请中断，让CPU执行中断函数）、事件响应（当外部中断检测到引脚变化时，中断信号就不会通向CPU了，而是通到其他外设，用来触发其他外设的操作（ADC,DMA）)。

### EXTI 结构框图

<img src="https://img2023.cnblogs.com/blog/3157802/202311/3157802-20231123091738881-1338910306.png" alt="img" style="zoom: 33%;" />



![image-20260811235335463](https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image-20260811235335463.png)

EXTI可分为两大部分功能，一个是产生中断，另一个是产生事件，这两个功能从硬件上就有所不同。产生中断线路目的是把输入信号输入到NVIC，进一步会运行中断服务函数实现功能，是软件级的。而产生事件线路目的就是传输一个脉冲信号给其他外设使用， 并且是电路级别的信号传输，属于硬件级的。

*另外，EXTI是在APB2总线上。*

### 中断/事件线

EXTI有20个中断/事件线，EXTI0~EXTI15用于GPIO，每个GPIO都可以被设为EXTI的输入源， 还有另外4根用于特定的外设事件。

<img src="C:/Users/acer/AppData/Roaming/Typora/typora-user-images/image-20260830091941940.png" alt="image-20260830091941940" style="zoom: 50%;" />

EXTI0至EXTI15用于GPIO，通过编程控制可以实现任意一个GPIO作为EXTI的输入源。EXTI0可以通过AFIO的外部中断配置寄存器1(AFIO_EXTICR1)的EXTI0[3:0]位选择配置为PA0、PB0、PC0、PD0、PE0、PF0、PG0、PH0或者PI0，其他EXTI线(EXTI中断/事件线)使用配置都是类似的。

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image25.png" alt="图 17‑2 EXTI0输入源选择" style="zoom:50%;" />

### EXTI初始化

HAL库函数的EXIT初始化非常简单，只需使能GPIO时钟并配置好IO口的模式，然后配置中断源、中断优先级、使能中断，最后编写实现中断服务函数。

```c
/**********HAL库NVIC函数**********/
HAL_NVIC_SetPriorityGrouping(IRQn_Type IRQn, uint32_t PreemptPriority, uint32_t SubPriority);	//HAL库中断优先级分组设置
HAL_NVIC_SetPriority(IRQn_Type IRQn, uint32_t PreemptPriority, uint32_t SubPriority);	//选择EXIT中断/事件线，设置中断线优先级
HAL_NVIC_EnableIRQ(IRQn_Type IRQn);	//使能指定的中断

// 中断入口（启动文件定义，自行实现）
void EXTI0_IRQHandler(void);
...
void EXTI15_10_IRQHandler(void);

// HAL库通用处理函数
void HAL_GPIO_EXTI_IRQHandler(uint16_t GPIO_Pin);

// 中断回调（需用户实现）
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin);
```

## 四、实验：外部中断控制

> 以F103-MINI开发板为例

### 硬件设计

轻触按键在按下时会使得引脚接通，通过电路设计可以使得按下时产生电平变化，

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image3a.png" alt="图 17‑3a MINI板、指南者和霸道开发板的按键原理图" style="zoom: 67%;" />

### 软件设计

1. 使能相关GPIO端口时钟；
2. 初始化用来产生中断的GPIO；
3. 初始化EXTI；
4. 配置NVIC；
5. 编写中断服务函数；

#### 软件分析

1. 按键和EXTI宏定义

```Markdown
/********************引脚定义******************************/
 #define KEY1_INT_GPIO_PORT                GPIOA
 #define KEY1_INT_GPIO_CLK_ENABLE()        __GPIOA_CLK_ENABLE();
 #define KEY1_INT_GPIO_PIN                 GPIO_PIN_0
 
 #define KEY1_INT_EXTI_IRQ                 EXTI0_IRQn
 #define KEY1_IRQHandler                   EXTI0_IRQHandler

 #define KEY2_INT_GPIO_PORT                GPIOC
 #define KEY2_INT_GPIO_CLK_ENABLE()        __GPIOA_CLK_ENABLE();
 #define KEY2_INT_GPIO_PIN                 GPIO_PIN_13
 
 #define KEY2_INT_EXTI_IRQ                 EXTI15_10_IRQn
 #define KEY2_IRQHandler                   EXTI15_10_IRQHandler
 /*******************************************************/
```

2. EXTI中断配置

```JavaScript
void EXTI_Key_Config(void)
 {
     GPIO_InitTypeDef GPIO_InitStructure;

     /*开启按键GPIO口的时钟*/
     KEY1_INT_GPIO_CLK_ENABLE();
     KEY2_INT_GPIO_CLK_ENABLE();

     /****** GPIO按键1的初始化 ******/
     GPIO_InitStructure.Pin = KEY1_INT_GPIO_PIN; 	/* 选择按键1的引脚 */
     GPIO_InitStructure.Mode = GPIO_MODE_IT_RISING; /* 设置引脚上升沿中断 */
     GPIO_InitStructure.Pull = GPIO_NOPULL; 		/* 设置引脚不上拉也不下拉 */
     HAL_GPIO_Init(KEY1_INT_GPIO_PORT, &GPIO_InitStructure); 
     
     /****** EXTI中断配置 中断源按键1引脚 ******/
     HAL_NVIC_SetPriority(KEY1_INT_EXTI_IRQ, 0, 0); /* 配置 EXTI中断源到key1 引脚、配置中断优先级*/
     HAL_NVIC_EnableIRQ(KEY1_INT_EXTI_IRQ);    		/* 使能中断，设置中断处理函数 */

    //同上
     GPIO_InitStructure.Pin = KEY2_INT_GPIO_PIN;
     GPIO_InitStructure.Mode = GPIO_MODE_IT_FALLING;
     HAL_GPIO_Init(KEY2_INT_GPIO_PORT, &GPIO_InitStructure);
     HAL_NVIC_SetPriority(KEY2_INT_EXTI_IRQ, 0, 0);
     HAL_NVIC_EnableIRQ(KEY2_INT_EXTI_IRQ);
 }
```

3. EXTI中断服务函数

```Java
//__HAL_GPIO_EXTI_GET_IT函数用来获取EXTI的中断标志位状态，如果EXTI线有中断发生函数返回“SET”，否则返回“RESET”。
void KEY1_IRQHandler(void)
 {
     //确保是否产生了EXTI Line中断
     if (__HAL_GPIO_EXTI_GET_IT(KEY1_INT_GPIO_PIN) != RESET) {
         LED1_TOGGLE;								 // LED1 取反
         __HAL_GPIO_EXTI_CLEAR_IT(KEY1_INT_GPIO_PIN);	// 清除中断标志位
     }
 }

 void KEY2_IRQHandler(void)
 {
     //确保是否产生了EXTI Line中断
     if (__HAL_GPIO_EXTI_GET_IT(KEY2_INT_GPIO_PIN) != RESET) {
         LED2_TOGGLE;							    // LED2 取反
         __HAL_GPIO_EXTI_CLEAR_IT(KEY2_INT_GPIO_PIN); // 清除中断标志位
     }
 }
```

4. 主函数

```Java
int main(void)
 {
     SystemClock_Config(); /* 系统时钟初始化成72MHz */
     LED_GPIO_Config(); /* LED 端口初始化 */

     // 初始化EXTI中断，按下按键会触发中断，KEY1_IRQHandler和KEY2_IRQHandler处理中断，反转LED灯。
     EXTI_Key_Config();

     /* 等待中断，由于使用中断方式，CPU不用轮询按键 */
     while (1) {}
 }
```
