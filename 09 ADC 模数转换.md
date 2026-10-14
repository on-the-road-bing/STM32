# ADC 模数转换器

- ADC（Analog-Digital Converter）模拟-数字转换器，用于将引脚上连续变化的模拟电压转换为内存中存储的数字变量，建立模拟电路到数字电路的桥梁。
- 采集通道：18个输入通道 =16个外部通道 + 2个内部通道。
- 转换顺序：规则组和注入组两个转换单元。
- STM32f103 系列有3个ADC，精度12位，每个ADC最多有16个外部通道。其中ADC1、ADC2有16个外部通道，ADC3根据CPU引脚的不同通道数也不同。

## ADC功能框图剖析

> 掌握了ADC的功能框图，就可以对ADC有一个整体的把握，编程时候可以做到了然如胸。框图讲解采用从左到右的方式，跟ADC采集数据，转换数据，传输数据的方向大概一致。

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/938672a6bde74bfd988417ddf9f18112.png" alt="938672a6bde74bfd988417ddf9f18112.png" style="zoom: 50%;" />

![图 29‑1 单个ADC功能框图](https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image212.png)

1. 电压输入范围

   ADC的输入电压范围为：0~3.3V，转换结果范围：0~4095。

2. ADC通道

   ADC输入通道共18个：16个外部通道+2个内部通道，16个外部通道对应不同的GPIO口。

   - 外部通道：框图中的ADCx_IN0、ADCx_IN1…ADCx_IN15，16个通道对应着不同的IO口。

   - 内部通道：ADC1的模拟通道16连接到了芯片内的温度传感器，通道17连接Vrefint 。ADC2的模拟通道16和17连接到了内部的VSS。

   为了高效地管理多个采集任务，ADC通道被划分为两个功能不同的组：规则通道组、注入通道组。16个外部通道在转换时又分为规则通道和注入通道，其中规则通道最多16路，注入通道最多有4路。规则通道和注入通道是ADC的两套独立转换序列，核心区别在于优先级和响应机制。

   - 规则通道组：最常用的组，用于处理常规的、顺序的转换任务。一个ADC控制器最多可支持16个规则通道。转换完成后，数据会存入一个共享的规则数据寄存器中，因此需要及时读取，否则新数据会覆盖旧数据。
   - 注入通道组：高优先级通道组，最多可配置4个通道。它像一个“中断”任务，可以打断当前规则通道组的转换，优先处理紧急的模拟信号测量。转换完成后，数据保存在注入通道专用的寄存器中。

   <img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image310.png" alt="图 29‑2 STM32F103VET6 ADC 通道" style="zoom: 67%;" />

   | 特性       | 规则通道               | 注入通道                           |
   | ---------- | ---------------------- | ---------------------------------- |
   | 本质       | 常规排队采样           | VIP 插队采样                       |
   | 通道数量   | 最多 16 路             | 最多 4 路                          |
   | 优先级     | 低                     | 高，可打断规则通道                 |
   | 数据寄存器 | 共用 1 个 `ADC_DR`     | 独立 4 个 `ADC_JDR1~JDR4`          |
   | DMA 支持   | ✅ 支持                 | ❌ 不支持（需中断读取）             |
   | 触发方式   | 软件、定时器、外部触发 | 软件、定时器、外部触发（独立配置） |
   | 典型用途   | 周期性传感器采集       | 紧急事件、故障保护                 |


3. 转换顺序

   规则序列和注入序列是 STM32 ADC 的两套独立转换队列。

   - 规则序列：规则序列寄存器有3个：SQR3、SQR2、SQR1。通过 3 个序列寄存器定义最多 16 个通道的转换顺序。

   - 注入序列：注入序列寄存器JSQR只有一个，最多支持4个通道，具体多少个由JSQR的JL[1:0]决定。

     ![image-20260823125938949](https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image-20260823125938949.png)

4. 触发源

   提供开始转换的START信号，决定了转换何时启动，可以分别选择不同的触发事件。

   - 触发源分类

     - 软件触发：通过代码置位启动转换，典型应用有调试、低速单点采集。

     - 硬件触发：由定时器、外部引脚等事件自动启动。

   - 不同类型的触发源配置

     STM32F1 系列（经典3位选择），配置集中在 ADC_CR2 寄存器：

     | 位域                       | 功能                                                         |
     | :------------------------- | :----------------------------------------------------------- |
     | `ADON` (Bit 0)             | STM32 F1系列特有：置 1 同时使能 ADC 并启动一次规则组转换（软件触发） |
     | `EXTTRIG` (Bit 22)         | 规则组外部触发使能                                           |
     | `EXTSEL[2:0]` (Bit 23-25)  | 规则组触发源选择（8 种）                                     |
     | `JEXTTRIG` (Bit 23)        | 注入组外部触发使能                                           |
     | `JEXTSEL[2:0]` (Bit 24-26) | 注入组触发源选择（8 种）                                     |
   
5. 转换时间

   - ADC时钟

     ADC模块挂载在APB2总线上，ADC_CLK由PCLK2经过分频产生，分频因子2/4/6/8分频。

   - 采样时间

     ADC使用若干个ADC_CLK周期对输入电压进行采样，采样周期数通过ADC 采样时间寄存器设置。

     每个通道独立设置采样时间，其中最小采样周期是1.5个，即如果要达到最快的采样，那应该设置采样周期为1.5个周期，这里说的周期就是1/ADC_CLK。

   - ADC转换时间
   
     转换公式：`Tconv = 采样时间 + 12.5个周期`。采样时间：由配置决定。12.5个周期：是ADC进行12位逐次逼近转换所需的固定时间。
   

## ADC的四大工作模式

> ADC初始化的结构体里有两个参数，一个是选择单次转换还是连续转换，另一个是选择扫描模式还是非扫描模式，通过组合使用，可以应对不同的应用场景：

- 单次转换模式：每触发一次，ADC仅完成一个通道的一次转换，然后停止。适用于对某个信号进行单次读取。
- 连续转换模式：一旦触发，ADC将不间断地对同一个通道进行连续转换。适用于需要连续监测单一信号的应用。
- 扫描模式：此模式用于多通道采集。触发一次后，ADC会按照预设的顺序，自动依次转换所有已启用的通道。
- 间断模式：在扫描模式的基础上，每次触发只转换序列中的一个通道，需要多次触发才能完成所有通道的转换。可以看作是“手动挡”的扫描模式。

## ADC采集方式

根据应用场景，可以选择以下三种方式来实现ADC采集

- 方式一：轮询方式 (Polling) - 最简单

- 方式二：中断方式 (Interrupt) - 不阻塞CPU

  ADC转换完成后触发中断，在中断回调函数中读取数据。分为转换结束中断（规则、注入通道序列转换完成）、模拟看门狗中断。

- 方式三：DMA方式 (DMA) - 最高效，多通道采集首选。

  ADC每完成一次转换，DMA都会自动将数据从ADC的数据寄存器搬运到指定的内存数组中，完全无需CPU干预。

## 电压转换

模拟电压经过ADC转换后，是一个12位的数字值，如果通过串口以16进制打印出来的话，可读性比较差，那么有时候就需要把数字电压转换成模拟电压，也可以跟实际的模拟电压（用万用表测）对比，看看转换是否准确。

一般在设计原理图的时候会把ADC的输入电压范围设定在：0~3.3v，因为ADC是12位的，那么12位满量程对应的就是3.3V，12位满量程对应的数字值是：2^12。数值0对应的就是0V。如果转换后的数值为 X ，X对应的模拟电压为Y，那么会有这么一个等式成立： 2^12 / 3.3 = X / Y。

## ADC相关结构体与函数

1. **ADC初始化结构体**

> HAL库函数对每个外设都建立了一个初始化结构体 xxx_InitTypeDef(xxx为外设名称)，结构体成员用于设置外设工作参数，并由HAL库函数xxx_Init()调用这些设定参数进入设置外设相应的寄存器，达到配置外设工作环境的目的。

```C
//ADC初始化参数结构体
typedef struct
{
  uint32_t DataAlign;			//输出数据对齐方式
  uint32_t ScanConvMode; 		//扫描模式ADC。ENABLE：按NbrOfConversion指定的通道顺序转换；DISABLE：单通道模式
  uint32_t ContinuousConvMode;	// 连续转换模式。ENABLE：触发一次后连续不断地转换；DISABLE：单次转换（转换完即停）
  uint32_t NbrOfConversion;		//规则转换序列数目。1~16，仅在扫描模式使能时有效
  uint32_t DiscontinuousConvMode;//不连续转换模式。ENABLE：将扫描序列分成若干子组（每次触发只转换一组）；DISABLE：完整序列一次转换完
  uint32_t NbrOfDiscConversion;	//不连续转换的通道数。1~8，仅在间断模式使能时有效
  uint32_t ExternalTrigConv;	//ADC转换触发信号选择
}ADC_InitTypeDef;

//ADC句柄
typedef struct
{
  ADC_TypeDef       *Instance;    //寄存器基地址指针
  ADC_InitTypeDef   Init;         //ADC初始化参数结构体
  DMA_HandleTypeDef	*DMA_Handle;  //DMA处理程序指针
  HAL_LockTypeDef	Lock;         //ADC锁定对象
  __IO uint32_t		State;        //ADC通信状态
  __IO uint32_t		ErrorCode;    //ADC错误码
}ADC_HandleTypeDef;
```

2. **ADC 通道配置结构体**

> HAL 库中配置单个 ADC 通道的核心结构体。主要负责设定一个通道的转换通道、转换顺序、采样时间等独立参数，并通过 `HAL_ADC_ConfigChannel()` 函数生效

```c
typedef struct
{
  uint32_t Channel;        /* ADC转换通道选择 */
  uint32_t Rank;           /* 转换顺序（第几个转换） */
  uint32_t SamplingTime;   /* 采样周期 */
  uint32_t Offset;         /* 预留，未用到 */
} ADC_ChannelConfTypeDef;

HAL_ADC_ConfigChannel();
```

3.库函数

```c
HAL_StatusTypeDef HAL_ADC_Init(ADC_HandleTypeDef* hadc);
HAL_StatusTypeDef HAL_ADC_ConfigChannel(ADC_HandleTypeDef* hadc, ADC_ChannelConfTypeDef* sConfig);
HAL_StatusTypeDef HAL_ADC_Start_IT(ADC_HandleTypeDef* hadc);
```

## 实验1：独立模式单通道采集

比较基础实用的单通道采集，实现开发板上电位器电压的采集，并通过串口打印至PC端串口调试助手。单通道采集适用AD转换完成中断，在中断服务函数中读取数据，不使用DMA传输，在多通道采集时才使用DMA传输。

### 硬件设计

MINI、指南者和霸道开发板板载一个贴片滑动变阻器，滑动变阻器的滑片端连接到了STM32的PC1引脚，电路设计见图

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image66.png" alt="图 29‑5 开发板电位器部分原理图" style="zoom: 50%;" />

贴片滑动变阻器的动触点连接至STM32芯片的ADC通道引脚。当旋转滑动变阻器调节旋钮时，其动触点电压也会随之改变，电压变化范围为0~3.3V，亦是开发板默认的ADC电压采集范围。

### 软件设计

1. 初始化ADC用到的GPIO：模拟输入；
2. 设置ADC外设时钟；
3. 设置ADC的工作参数并初始化；
4. 设置ADC转换通道顺序、分频因子、采样时间；
5. 设置ADC中断优先级，开启ADC转换完成中断，在中断内读取转换完数据；

#### 代码分析

1. ADC宏定义

   > 使用宏定义引脚信息方便硬件电路改动时程序移植。

   ```c
   // ADC GPIO 宏定义
   #define RHEOSTAT_ADC_GPIO_PORT              GPIOC
   #define RHEOSTAT_ADC_GPIO_PIN               GPIO_PIN_1
   #define RHEOSTAT_ADC_GPIO_CLK_ENABLE()      __HAL_RCC_GPIOC_CLK_ENABLE()
   // ADC 序号宏定义
   #define RHEOSTAT_ADC                        ADC1
   #define RHEOSTAT_ADC_CLK_ENABLE()           __HAL_RCC_ADC1_CLK_ENABLE();
   #define RHEOSTAT_ADC_CHANNEL                ADC_CHANNEL_11
   ```
   
2. ADC GPIO初始化函数

   使用到GPIO时必须开启对应的GPIO时钟，GPIO用于ADC转换功能必须配置为模拟输入模式。

   ```C
   static void Rheostat_ADC_GPIO_Config(void)
    {
        GPIO_InitTypeDef GPIO_InitStructure;
        RHEOSTAT_ADC_GPIO_CLK_ENABLE();				// 使能 GPIO 时钟
   
        // 配置 GPIO
        GPIO_InitStructure.Pin = RHEOSTAT_ADC_GPIO_PIN; //指定引脚
        GPIO_InitStructure.Mode = GPIO_MODE_ANALOG; 	//模拟输入模式
        GPIO_InitStructure.Pull = GPIO_NOPULL ; 		//不上拉不下拉
        HAL_GPIO_Init(RHEOSTAT_ADC_GPIO_PORT, &GPIO_InitStructure);
    }
   ```
   
3. 配置ADC工作模式

   使用ADC_HandleTypeDef和ADC_ChannelConfTypeDef结构体分别定义ADC初始化和ADC通道配置变量。

   ```c
   static void Rheostat_ADC_Mode_Config(void)
    {
        RHEOSTAT_ADC_CLK_ENABLE(); // 开启ADC时钟
       
        /*-------------------ADC Init 结构体参数初始化-----------------------*/
        ADC_Handle.Instance = RHEOSTAT_ADC;
        ADC_Handle.Init.ClockPrescaler = ADC_CLOCKPRESCALER_PCLK_DIV4; // 时钟为fpclk 4分频
        ADC_Handle.Init.Resolution = ADC_RESOLUTION_12B; 	// ADC 分辨率
        ADC_Handle.Init.ScanConvMode = DISABLE; 		    // 禁止扫描模式，多通道采集才需要
        ADC_Handle.Init.ContinuousConvMode = ENABLE; 		// 连续转换
        ADC_Handle.Init.DiscontinuousConvMode = DISABLE; 	// 非连续转换
        ADC_Handle.Init.NbrOfDiscConversion   = 0; 		// 非连续转换个数
        ADC_Handle.Init.ExternalTrigConvEdge = ADC_EXTERNALTRIGCONVEDGE_NONE; //禁止外部边沿触发
        //ADC_Handle.Init.ExternalTrigConv   = ADC_EXTERNALTRIGCONV_T1_CC1; //使用软件触发，外部触发不用配置，注释掉即可
        ADC_Handle.Init.DataAlign = ADC_DATAALIGN_RIGHT; 	//数据右对齐
        ADC_Handle.Init.NbrOfConversion = 1; 			   //转换通道 1个
        ADC_Handle.Init.DMAContinuousRequests = ENABLE;  	//使能连续转换请求
        ADC_Handle.Init.EOCSelection          = DISABLE; 	//转换完成标志
        HAL_ADC_Init(&ADC_Handle); 					  //初始化ADC
        /*------------------------------------------------------------------*/
        
        /*-------------------配置ADC通道参数-----------------------*/
        //通道转换顺序为1，第一个转换，采样时间为3个时钟周期
        ADC_Config.Channel      = RHEOSTAT_ADC_CHANNEL; 	// 电位器连接的 ADC 通道
        ADC_Config.Rank         = 1;				   	   // 序列排名,第 1 个转换
        ADC_Config.SamplingTime = ADC_SAMPLETIME_56CYCLES; // 采样时间间隔
        ADC_Config.Offset       = 0;					  // 无数据偏移
        HAL_ADC_ConfigChannel(&ADC_Handle, &ADC_Config); 
        HAL_ADC_Start_IT(&ADC_Handle);		// 以中断方式启动 ADC（转换完成后进入 ADC_IRQHandler）
    }
   ```

4. 配置中断配置

```c
static void Rheostat_ADC_NVIC_Config(void)
{
     HAL_NVIC_SetPriority(Rheostat_ADC_IRQ, 0, 0);
     HAL_NVIC_EnableIRQ(Rheostat_ADC_IRQ);
 }
```

5. ADC中断服务函数

```C
 void ADC_IRQHandler(void)
 {
     HAL_ADC_IRQHandler(&ADC_Handle);
 }
 //转换完成中断回调函数（非阻塞模式）
 void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef* AdcHandle)
 {
     ADC_ConvertedValue = HAL_ADC_GetValue(AdcHandle);	     // 获取结果
 }
```

> HAL_ADC_IRQHandler是HAL中自带的一个中断服务函数，他处理过程中会指向一个回调函数给我们去添加用户代码，这里使用HAL_ADC_ConvCpltCallback转换完成中断，在ADC转换完成后就会进入中断服务函数，再进入回调函数，在回调函数内直接读取ADC转换结果保存在变量ADC_ConvertedValue中。
>
> ADC_GetConversionValue函数是获取ADC转换结果值的库函数，只有一个形参为ADC句柄，该函数还返回一个16位的ADC转换结果值

6. 主函数

```C
int main(void)
{
    SystemClock_Config();	// 配置系统时钟为72 MHz
    UARTx_Config();	 	   // 配置调试串口相关参数，配置模式为 115200 8-N-1
    Rheostat_Init();
    while (1) {
        ADC_Vol =(float)ADC_ConvertedValue/4096*(float)3.3; // 读取转换的AD值
        printf("\r\n The current AD value = 0x%04X \r\n", ADC_ConvertedValue);
        printf("\r\n The current AD value = %f V \r\n",ADC_Vol);
        Delay(0x8fffff);
    }
}
```

## 实验2：独立模式多通道采集

## 实验3：双重ADC同步规则模式采集
