# 03 GPIO

## 一、GPIO简介

- GPIO（General Purpose Input Output）通用输入输出口
- 可配置为8种输入输出模式
- 引脚电平：0V~3.3V，部分引脚可容忍5V
- 输出模式下可控制端口输出高低电平，用以驱动LED、控制蜂鸣器、模拟通信协议输出时序等
- 输入模式下可读取端口的高低电平或电压，用于读取按键输入、外接模块电平信号输入、ADC电压采集、模拟通信协议接收数据等
- 所有GPIO都挂在APB2外设总线上，GPIO被分成很多组（GPIOA、GPIOB、\.\.\.\.\.\.），每组16个引脚（也可能少于16个），GPIOA的0号引脚也称为PA0，GPIOA的1号引脚PA1，以此类推。

### 1. GPIO基本结构

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/5dc4343e2cfa4da0a22805e6a6d53908.png" alt="5dc4343e2cfa4da0a22805e6a6d53908.png" style="zoom:50%;" />

首先所有GPIO都挂在APB2外设总线上，名称按GPIOA、GPIOB、GPIOC...命名，对于STM32F103每个GPIO外设有16个引脚编号0-15，GPIOA的0号引脚也称为PA0，GPIOA的1号引脚PA1，以此类推。

 每个GPIO模块，主要包含寄存器和驱动器，内核通过APB2总线对寄存器进行读写，完成输出电平和读取电平的功能。


### 2. GPIO硬件框图剖析

从最右端看起，最右端就是代表STM32芯片引出的GPIO引脚，其余部件都位于芯片内部。

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image 14.png" alt="image\.png" style="zoom: 80%;" />

### 3. GPIO工作模式

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/5c78ce0a646a4db284903c82853839f5.png" alt="5c78ce0a646a4db284903c82853839f5.png" style="zoom: 50%;" />

## 二、GPIO相关结构体

```C++
// GPIO寄存器结构体定义
 typedef struct
 {
     __IO uint32_t CRL;       // 端口配置低寄存器，     地址偏移0X00
     __IO uint32_t CRH;       // 端口配置高寄存器，     地址偏移0X04
     __IO uint32_t IDR;       // 端口数据输入寄存器，   地址偏移0X08
     __IO uint32_t ODR;       // 端口数据输出寄存器，   地址偏移0X0C
     __IO uint32_t BSRR;      // 端口位设置/清除寄存器，地址偏移0X10
     __IO uint32_t BRR;       // 端口位清除寄存器，     地址偏移0X14
     __IO uint32_t LCKR;      // 端口配置锁定寄存器，   地址偏移0X18
 } GPIO_TypeDef;
 
// GPIO初始化结构体
typedef struct
{
    uint32_t Pin;     		//选择配置的引脚,GPIO_PIN_0 ~ GPIO_PIN_15
    uint32_t Mode;    		//引脚工作模式
    uint32_t Pull;    		//引脚内部上下拉电阻配置
    uint32_t Speed;   		//引脚输出速度
    uint32_t Alternate;   	//复用功能编号AFx
} GPIO_InitTypeDef;
 
// GPIO工作模式枚举定义
typedef enum
{
    GPIO_Mode_AIN = 0x0,           // 模拟输入     (0000 0000)b
    GPIO_Mode_IN_FLOATING = 0x04,  // 浮空输入     (0000 0100)b
    GPIO_Mode_IPD = 0x28,          // 下拉输入     (0010 1000)b
    GPIO_Mode_IPU = 0x48,          // 上拉输入     (0100 1000)b

    GPIO_Mode_Out_OD = 0x14,       // 开漏输出     (0001 0100)b
    GPIO_Mode_Out_PP = 0x10,       // 推挽输出     (0001 0000)b
    GPIO_Mode_AF_OD = 0x1C,        // 复用开漏输出  (0001 1100)b
    GPIO_Mode_AF_PP = 0x18         // 复用推挽输出  (0001 1000)b
} GPIOMode_TypeDef;
```

## 三、GPIO应用-输出点灯

> F103-MINI开发板为例

### 硬件设计

![F103-MINI开发板的LED硬件原理图](https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image22.png)

### 软件设计

1. 使用RCC使能GPIO外设时钟；
2. 初始化GPIO结构体变量，目标引脚为推挽输出模式；
3. 编写简单测试程序，控制GPIO引脚输出高、低电平。

#### 代码分析

LED灯引脚宏定义

```c
/*********************引脚定义*********************/
#define LED1_PIN                  GPIO_PIN_2
#define LED1_GPIO_PORT            GPIOC
#define LED1_GPIO_CLK_ENABLE()   __HAL_RCC_GPIOC_CLK_ENABLE()

#define LED2_PIN                  GPIO_PIN_3
#define LED2_GPIO_PORT            GPIOC
#define LED2_GPIO_CLK_ENABLE()   __HAL_RCC_GPIOC_CLK_ENABLE()
```

控制LED灯亮灭状态的宏定义

> 为了方便控制LED灯，把LED灯常用的亮、灭及状态反转的控制也直接定义成宏

```C++
 // 控制LED灯亮灭的宏，LED低电平亮，设置ON=0，OFF=1
#define ON  GPIO_PIN_RESET
#define OFF GPIO_PIN_SET

/* 带参宏，可以像内联函数一样使用 */
#define LED1(a) HAL_GPIO_WritePin(LED1_GPIO_PORT,LED1_PIN,a)
#define LED2(a) HAL_GPIO_WritePin(LED2_GPIO_PORT,LED2_PIN,a)

/* 直接操作寄存器的方法控制IO */
#define digitalHi(p,i)      {p->BSRR=i;}        //设置为高电平
#define digitalLo(p,i)      {p->BSRR=(uint32_t)i << 16;}        //输出低电平
#define digitalToggle(p,i)  {p->ODR ^=i;}     //输出反转状态

/* 定义控制IO的宏 */
#define LED1_TOGGLE   digitalToggle(LED1_GPIO_PORT,LED1_PIN)
#define LED1_OFF      digitalHi(LED1_GPIO_PORT,LED1_PIN)
#define LED1_ON       digitalLo(LED1_GPIO_PORT,LED1_PIN)

#define LED2_TOGGLE   digitalToggle(LED2_GPIO_PORT,LED2_PIN)
#define LED2_OFF      digitalHi(LED2_GPIO_PORT,LED2_PIN)
#define LED2_ON       digitalLo(LED2_GPIO_PORT,LED2_PIN)
```

LED GPIO初始化函数

```c
void LED_GPIO_Config(void)
{
    GPIO_InitTypeDef  GPIO_InitStruct;				//定义一个GPIO_InitTypeDef类型的结构体             
    /*开启LED相关的GPIO外设时钟*/
    LED1_GPIO_CLK_ENABLE();
    LED2_GPIO_CLK_ENABLE();

    GPIO_InitStruct.Pin = LED1_PIN; 				//选择要控制的GPIO引脚
    GPIO_InitStruct.Mode  = GPIO_MODE_OUTPUT_PP; 	 //设置引脚的输出类型为推挽输出
    GPIO_InitStruct.Pull  = GPIO_PULLUP;			//设置引脚为上拉模式
    GPIO_InitStruct.Speed = GPIO_SPEED_HIGH;		//设置引脚速率为高速
    HAL_GPIO_Init(LED1_GPIO_PORT, &GPIO_InitStruct); //调用库函数初始化GPIO

    GPIO_InitStruct.Pin = LED2_PIN;
    HAL_GPIO_Init(LED2_GPIO_PORT, &GPIO_InitStruct);

    LED_RGBOFF;									  //关闭RGB灯
}
```

主函数

```c
int main(void)
{
    SystemClock_Config(); 	//系统时钟初始化成72MHz
    LED_GPIO_Config();    	//LED 端口初始化

    /* 控制LED灯 */
    while (1) {
        LED1( ON );      	// 亮
        HAL_Delay(1000);
        LED1( OFF );      	// 灭
        HAL_Delay(1000);

        LED2( ON );     	// 亮
        HAL_Delay(1000);
        LED2( OFF );      	// 灭
    }
}
```

## 四、GPIO应用-输入按键检测

> 检测按键的状态，实现按键控制LED灯。
> F103-MINI开发板为例

### 硬件设计

<img src="https://doc.embedfire.com/mcu/stm32/f103/hal_general/zh/latest/_images/image2a.png" alt="图 13‑2a MINI板、指南者和霸道开发板的按键原理图" style="zoom:67%;" />

从按键的原理图可知，按键在没有被按下的时候，GPIO引脚的输入状态为低电平，当按键按下时，GPIO引脚的输入状态为高电平。只要检测引脚的输入电平，即可判断按键是否被按下。

### 软件设计

1. 使能GPIO端口时钟；

2. 初始化GPIO目标引脚为输入模式\(浮空输入\)；

3. 编写简单测试程序，检测按键的状态，实现按键控制LED灯。

#### 代码分析

按键引脚宏定义

```c
/*********************引脚定义*************************/
 #define KEY1_PIN                  GPIO_PIN_0
 #define KEY1_GPIO_PORT            GPIOA
 #define KEY1_GPIO_CLK_ENABLE()    __GPIOA_CLK_ENABLE()

 #define KEY2_PIN                  GPIO_PIN_13
 #define KEY2_GPIO_PORT            GPIOC
 #define KEY2_GPIO_CLK_ENABLE()    __GPIOC_CLK_ENABLE()
 /****************************************************/

 //按键按下标置宏：按键按下为高电平，设置 KEY_ON=1， KEY_OFF=0
 #define KEY_ON  1
 #define KEY_OFF 0
```

按键 GPIO初始化函数

```c
void Key_GPIO_Config(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    /*开启按键GPIO口的时钟*/
    KEY1_GPIO_CLK_ENABLE();
    KEY2_GPIO_CLK_ENABLE();

    GPIO_InitStructure.Pin = KEY1_PIN; 				  /*选择按键的引脚*/
    GPIO_InitStructure.Mode = GPIO_MODE_INPUT; 		   /*设置引脚为输入模式*/
    GPIO_InitStructure.Pull = GPIO_NOPULL; 			   /*设置引脚不上拉也不下拉*/
    HAL_GPIO_Init(KEY1_GPIO_PORT, &GPIO_InitStructure); /*初始化GPIO按键引脚*/

    GPIO_InitStructure.Pin = KEY2_PIN;
    HAL_GPIO_Init(KEY2_GPIO_PORT, &GPIO_InitStructure);
}
```

检测按键的状态

> 初始化按键后，通过检测对应引脚的电平来判断按键状态。

```c
uint8_t Key_Scan(GPIO_TypeDef* GPIOx,uint16_t GPIO_Pin)
 {
     /*检测是否有按键按下：若检测到按键按下，则使用while循环持续检测按键状态， 直到按键释放，按键释放后Key_Scan函数返回一个“KEY_ON”值；若没有检测到按键按下，则函数直接返回“KEY_OFF”。*/
     if (HAL_GPIO_ReadPin(GPIOx,GPIO_Pin) == KEY_ON ) {
         /*等待按键释放 */
         while (HAL_GPIO_ReadPin(GPIOx,GPIO_Pin) == KEY_ON);
         return  KEY_ON;
     } else
         return KEY_OFF;
 }
```

主函数

```C
int main(void)
 {
     SystemClock_Config(); /* 系统时钟初始化成72MHz */
     LED_GPIO_Config(); /* LED 端口初始化 */
     Key_GPIO_Config(); /*初始化按键*/
    
     /* 轮询按键状态，若按键按下则反转LED */
     while (1) {
         if ( Key_Scan(KEY1_GPIO_PORT,KEY1_PIN) == KEY_ON  ) 
             LED1_TOGGLE;    /*LED1反转*/
         if ( Key_Scan(KEY2_GPIO_PORT,KEY2_PIN) == KEY_ON  ) 
             LED2_TOGGLE;    /*LED2反转*/
     }
 }
```

