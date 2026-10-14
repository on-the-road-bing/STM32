# PWR 电源控制——实现低功耗

STM32微控制器广泛应用于各种嵌入式系统中，电源管理是其核心功能之一。有效的电源控制不仅可以延长电池寿命，还能提高系统的稳定性和可靠性。

## 一、PWR简介

- PWR（Power Control）电源控制
- 负责管理STM32内部的电源供电部分，可实现可编程电压监测器、低功耗模式的功能。
- 可编程电压监测器（PVD）可以监控VDD电源电压，当VDD下降到PVD阀值以下或上升到PVD阀值之上时，PVD会触发中断，用于执行紧急处理任务。
- 低功耗模式包括睡眠模式（Sleep）、停机模式（Stop）和待机模式（Standby），可在系统空闲时，降低STM32的功耗，延长设备使用时间。

## 二、电源监控器

> STM32芯片主要通过引脚 VDD 从外部获取电源，在它的内部具有电源监控器用于检测 VDD 的 电压，以实现复位功能及掉电紧急处理功能，保证系统可靠地运行。

### 1. 上电复位与掉电复位 

上电复位与掉电复位 (POR与PDR)

当检测到 VDD 的电压低于阈值 VPOR 及 VPDR 时，无需外部电路辅助，STM32芯片会自动保持在复位状态，防止因电压不足强行工作而带来严重的后果。

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image-20260828103127416.png" alt="image-20260828103127416" style="zoom:33%;" />

上图所示，在刚开始电压低于 VPOR 时 (约 1.92V)，STM32 保持在上电复位状态 (POR，Power On Reset)，当 VDD 电压持续 上升至大于 VPOR 时，芯片开始正常运行，而在芯片正常运行的时候，当检测到 VDD 电压下降 至低于 VPDR 阈值 (约 1.88V)，会进入掉电复位状态 (PDR，Power Down Reset)。

### 2. 可编程电压检测器 PVD

STM32还提供了可编程的电压监测器PVD，它也是实时检测VDD的电压，当检测到电压低于编程的VPVD阈值时，会向内核产生 一个PVD中断 (EXTI16 线中断) 以使内核在复位前进行紧急处理。该电压阈值可通过电源控制寄存器 PWR_CSR 设置。

PVD可配置8个等级，其中上升沿和下降沿分别表示类似图POR与PDR中VDD电压上升过程及下降过程的阈值。可使用库函数 `PWR_PVDLevelConfig()` 配置阈值等级。

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image-20260828103534940.png" alt="image-20260828103534940" style="zoom: 25%;" />

```C
//可编程电压检测器PVD初始化结构体
typedef struct
{
  uint32_t PVDLevel;   //电压检测阈值
  uint32_t Mode;       //PVD触发模式
}PWR_PVDTypeDef;


PWR_PVDTypeDef pvdHandle;
void PVD_Init(void) {
    pvdHandle.PVDLevel = PWR_PVDLEVEL_7;		// 设定阈值为约 2.9V（高阈值，适合 3.3V 系统）
    pvdHandle.Mode = PWR_PVD_MODE_IT_FALLING;	// 仅在电压跌落（低于 2.9V）时触发中断，用于紧急报警
    HAL_PWR_ConfigPVD(&pvdHandle);			   // 将配置写入寄存器

    HAL_NVIC_SetPriority(PVD_IRQn, 0, 0);		// 使能PVD中断（在 NVIC 中使能）
    HAL_NVIC_EnableIRQ(PVD_IRQn);
    
    HAL_PWR_EnablePVD();					  // 硬件上开启 PVD 功能
}

// 中断服务函数（位于stm32f1xx_it.c）
void PVD_IRQHandler(void) {
    HAL_PWR_PVD_IRQHandler(); // 调用HAL的通用处理函数，它会清除标志位并调用回调
}

// 回调函数（用户实现，放在 main.c 或其他文件）
void HAL_PWR_PVDCallback(void) {
    // 电压跌落了！执行紧急操作
    // 例如：关闭电机、将关键变量存入备份寄存器、或者根本不进入低功耗模式
}
```



## 三、电源系统框图

为方便进行电源管理，STM32把外设、内核等模块根据功能划分了供电区域，其内部电源区域划分见图：

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image-20260825094325115.png" alt="image-20260825094325115" style="zoom:40%;" />

从框图可知，STM32的电源系统主要分为三类：数字电源、模拟电源、备份电源。

- 模拟电源：VDDA用于模拟部分的电源，比如ADC。
- 数字电源：VDD用于数字部分的电源，是其主电源。
- 备份电源：VBAT用于备份区域电源，比如RTC、备份SRAM等，一旦主电源断开，VBAT可以为这些区域提供电源。

## 四、低功耗模式

> 按功耗由高到低排列，STM32具有运行、睡眠、停止和待机四种工作模式。上电复位后 STM32 处于运行状态时，当内核不需要继续运行，就可以选择进入后面的三种低功耗模式降低功耗，这三种模式中，电源消耗不同、唤醒时间不同、唤醒源不同，用户需要根据应用需求，选择最佳的低功耗模式。三种低功耗的模式说明见表。
>
> <img src="https://i-blog.csdnimg.cn/blog_migrate/9ba63f34b434d02589ba07b4b91b7dbd.png" alt="img" style="zoom:50%;" />
>
> <img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image-20260825101606611.png" alt="image-20260825101606611" style="zoom:50%;" />

### 1. 睡眠模式

在睡眠模式中，仅关闭了内核时钟，内核停止运行，但其片上外设，CM3 核心的外设全都还照常运行。

有两种方式进入睡眠模式，进入方式决定了睡眠唤醒的方式，分别是WFI(wait for interrupt)和WFE(wait for event)，即由等待“中断”唤醒和由“事件”唤醒。

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/3e1eec9ae9995cbbc94e850fa8bfd11f.png" alt="img" style="zoom:50%;" />

```C
//等待中断，暂停至任意中断产生后被唤醒
 #define __WFI      __wfi
//等待事件，暂停至任意事件产生后被唤醒
 #define __WFE      __wfe

//HAL库控制进入睡眠模式
HAL_PWR_EnterSLEEPMode()
```

软件设计步骤

1. 进入睡眠模式前：执行必要的准备操作。
2. 暂停SysTick中断：`HAL_SuspendTick()`。如果不暂停，SysTick中断会持续唤醒CPU。
3. 执行睡眠指令：调用`HAL_PWR_EnterSLEEPMode()`。
4. 从睡眠模式唤醒后恢复SysTick中断。调用`HAL_ResumeTick()`。
5. 继续执行：CPU从进入睡眠模式的下一条指令开始继续运行。

### 2. 停止模式

在停止模式中，进一步关闭了其它所有的时钟，于是所有的外设都停止了工作，但由于其1.8V区域的部分电源没有关闭， 还保留了内核的寄存器、内存的信息，所以从停止模式唤醒，并重新开启时钟后，还可以从上次停止处继续执行代码。 

停止模式可以由任意一个外部中断（EXTI）唤醒，在停止模式中可以选择电压调节器为开模式或低功耗模式。

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/cfb07e1657b4f21a70e0208ac7381ea6.png" alt="img" style="zoom:50%;" />

```C
直接调用WFI和WFE指令可以进入睡眠模式，而进入停止模式则还需要在调用指令前设置一些寄存器位， HAL库把这部分的操作封装到 `HAL_PWR_EnterSTOPMode()` 函数中。
注意：进入停止模式后，STM32的所有I/O都保持在停止前的状态，而当它被唤醒时，STM32使用HSI作为系统时钟(8MHz)运行，由于系统时钟会影响很多外设的工作状态，所以一般在唤醒后会重新开启HSE，把系统时钟设置回原来的状态。
```

软件设计步骤：

1. 进入停止模式前：执行必要的准备操作。
2. 暂停SysTick中断：调用 `HAL_SuspendTick()`，防止被其频繁唤醒。
3. （可选）配置FLASH掉电：调用 `HAL_PWREx_EnableFlashPowerDown()`，进一步降低功耗。
4. 执行停止指令：调用 `HAL_PWR_EnterSTOPMode()` 进入停止模式。
5. 从停止模式唤醒后：
   - 恢复SysTick中断：`HAL_ResumeTick()`。
   - 重新配置系统时钟：这是最关键的一步！停止模式唤醒后，系统时钟会默认切换到 HSI (高速内部时钟)。如果之前使用的是HSE或PLL，必须重新配置才能恢复到全速运行状态。
   - 重新初始化外设：部分外设（如ADC、DMA）在停止模式下可能断电，唤醒后需要重新初始化。



### 3. 待机模式

待机模式，除了关闭所有的时钟，还把1.8V区域的电源也完全关闭了，也就是说，从待机模式唤醒后，由于没有之前代码的运行记录，只能对芯片复位，重新检测boot条件，从头开始执行程序。

它有四种唤醒方式，分别是WKUP(PA0)引脚的上升沿，RTC闹钟事件，NRST引脚的复位和IWDG(独立看门狗)复位。

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/1ce6f9d7393d3f499e5d9cfa7a311af9.png" alt="img" style="zoom: 50%;" />

```C
HAL库进入待机模式的函数 `HAL_PWR_EnterSTANDBYMode()`
注意：进入待机模式后，除了被使能了的用于唤醒的I/O，其余I/O都进入高阻态，而从待机模式唤醒后，相当于复位STM32芯片，程序重新从头开始执行。
```

软件设计步骤：

1. 进入待机模式前：执行必要的准备操作。
2. 配置唤醒源：调用 `HAL_PWR_EnableWakeUpPin()` 使能WKUP引脚，也可使是RTC闹钟事件作为唤醒源。
3. 清除唤醒标志：调用 `__HAL_PWR_CLEAR_FLAG(PWR_FLAG_WU)`。清除历史唤醒标志，确保能正常进入待机模式。
4. 暂停SysTick中断：调用 `HAL_SuspendTick()`。
5. 执行待机指令：调用 `HAL_PWR_EnterSTANDBYMode()`。
6. 从待机模式唤醒后（相当于系统复位）。
   - 唤醒源触发（如WKUP引脚上升沿、RTC闹钟、NRST复位等）后，芯片会执行上电复位，程序从`main()`函数重新开始执行。
   - 不需要恢复SysTick中断（因为已经复位，HAL_Init()会重新初始化）。
   - 不需要重新配置系统时钟（复位后默认使用HSI，`SystemClock_Config()`会在初始化阶段被调用）。
   - 必须检查复位标志是否为待机唤醒，以执行不同的初始化逻辑。调用`__HAL_PWR_GET_FLAG()`。

三大低功耗模式对比

| 特性         | 睡眠模式 (Sleep)     | 停止模式 (Stop)                        | 待机模式 (Standby)        |
| :----------- | :------------------- | :------------------------------------- | :------------------------ |
| CPU状态      | 停止                 | 停止                                   | 停止                      |
| 外设与时钟   | 所有外设时钟保持运行 | 所有时钟关闭，外设电源保持供电(1.2V域) | 所有时钟关闭 (1.2V域断电) |
| SRAM与寄存器 | 完整保留             | 完整保留                               | 全部丢失 (备份寄存器除外) |
| 功耗水平     | 较高 (毫安级)        | 较低 (微安级，~2.8μA)                  | 最低 (微安级，~2.2μA)     |
| 唤醒源       | 任意中断/事件        | 任一外部中断(EXTI)                     | WKUP引脚、RTC闹钟、复位等 |
| 唤醒时间     | 极快 (无延时)        | 较快 (< 6μs)                           | 最慢 (相当于系统复位)     |
| 唤醒后动作   | 立即执行下一条指令   | 继续执行后续代码，需重新配置时钟       | 系统复位，程序从头执行    |

## 五、PWR实验

### PVD电源监控实验

软件设计

1. 初始化PVD中断；
2. 设置PVD电压监控等级并使能PVD；
3. 编写PVD中断服务函数，处理紧急任务。

### 睡眠模式实验

软件设计

1. 初始化用于唤醒的中断按键；
2. 进入睡眠状态；
3. 使用按键中断唤醒芯片；

代码分析

main函数

```c
 /**
 * 实验说明：
 * 1.本程序中绿灯表示正常运行，红灯表示睡眠状态，蓝灯表示刚从睡眠状态被唤醒
 * 2.程序运行一段时间后自动进入睡眠状态，在睡眠状态下，可使用KEY1或KEY2唤醒
 * 3.本实验执行这样一个循环：亮绿灯(正常运行)->亮红灯(睡眠模式)->按KEY1或KEY2唤醒->亮蓝灯(刚被唤醒)
 */
int main(void)
 {
     SystemClock_Config();		 // 初始化系统时钟为72MHz
     LED_GPIO_Config();			 // 初始化LED
     DEBUG_USART_Config();		 // 初始化调试串口，一般为串口1
     EXTI_Key_Config();			 //初始化按键为中断模式，按下中断后会进入中断服务函数

     while (1) {
         /*********执行任务***************************/
         printf("\r\n STM32正常运行，亮绿灯\r\n");
         LED_GREEN;
         HAL_Delay(2000);
         /*****任务执行完毕，进入睡眠降低功耗***********/

         printf("\r\n 进入睡眠模式，亮红灯,按KEY1或KEY2按键可唤醒\r\n");
         LED_RED;				   //使用红灯指示，进入睡眠状态
         HAL_SuspendTick();         //暂停滴答时钟，防止通过滴答时钟中断唤
         HAL_PWR_EnterSLEEPMode(PWR_MAINREGULATOR_ON,PWR_SLEEPENTRY_WFI);	//进入睡眠模式
         
         //等待中断唤醒  K1或K2按键中断
         LED_BLUE;				   //被唤醒，亮蓝灯指示
         HAL_ResumeTick();		    //被唤醒后，恢复滴答时钟
         HAL_Delay(2000);

         printf("\r\n 已退出睡眠模式\r\n");
     }
 }
```

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image-20260809002202424.png" alt="image-20260809002202424" style="zoom: 67%;" />

**中断服务函数**

```c
void KEY1_IRQHandler(void)
 {
     HAL_GPIO_EXTI_IRQHandler(KEY1_INT_GPIO_PIN);
 }
 void KEY2_IRQHandler(void)
 {
     HAL_GPIO_EXTI_IRQHandler(KEY2_INT_GPIO_PIN);
 }
 void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
 {
     LED_BLUE;
     if (GPIO_Pin==KEY1_INT_GPIO_PIN)
         printf("\r\n KEY1 按键中断唤醒 \r\n");
     else if (GPIO_Pin==KEY2_INT_GPIO_PIN)
         printf("\r\n KEY2 按键中断唤醒 \r\n");
     else {
     }
 }
```

### 停止模式实验

硬件设计

软件设计

1. 初始化用于唤醒的中断按键；
2. 选择电压调节器的工作模式并进入停止状态；
3. 使用按键中断唤醒芯片；
4. 重启HSE时钟，使系统完全恢复停止前的状态。

<img src="C:/Users/acer/AppData/Roaming/Typora/typora-user-images/image-20260809162315502.png" alt="image-20260809162315502" style="zoom:55%;" />

### 待机模式实验

软件设计

1. 清除WUF标志位；
2. 使能WKUP唤醒功能；
   1. 进入待机状态。
