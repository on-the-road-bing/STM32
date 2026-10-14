# DMA 直接存储器存取

DMA\(Direct Memory Access\)—直接存储器存取，是单片机的一个外设，主要功能是用来搬数据，但是不需要占用CPU， 即传输数据的时候，CPU可以干其他的事情，好像是多线程一样。数据传输支持从外设到存储器或者存储器到存储器， 这里的存储器可以是SRAM或者是FLASH。

- DMA（Direct Memory Access）直接存储器存取

- DMA可支持外设（外设寄存器）和存储器（SRAM和Flash）、存储器和存储器之间的高速数据传输（数据从一个地址空间复制到另一个地址空间）。无须CPU干预，节省了CPU的资源

- 2个DMA控制器，共12个独立可配置的通道（请求）： DMA1（7个通道）， DMA2（5个通道），每个通道专门用来管理来自于一个或多个外设对存储器访问的请求。

- 每个通道都支持软件触发和特定硬件触发。

  > 如果DMA进行的是存储器到存储器的数据转运，那就需要软件触发，软件触发之后DMA就会一股脑地把这批数据以最快的速度全部转运完成。
  > 如果DMA进行的是外设到存储器的数据转运，就不能一股脑了，因为外设的数据是有一定时机的，这时要用硬件触发，比如转运ADC的数据，那就得ADC每个通道AD转换完成后，硬件触发一次之后再DMA转运，触发一次转运一次，这样数据才是正确的。

## DMA框图

DMA控制器独立于内核，属于一个单独的外设，结构比较简单。

- DMA总线用于访问各个存储器
- DMA内部的多个通道可以进行独立的数据转运。
- 仲裁器用于调度各个通道防止发生冲突。
- AHB从设备用于配置DMA参数。
- DMA请求用于硬件触发DMA的数据转运。 

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image-20260813090014313.png" alt="image-20260813090014313" style="zoom: 33%;" />

![img](https://gitee.com/yb_wang6/typora_imgs/raw/master/img/de801d9fae114310aeb661cc3fc601e8.png)

## 基本结构

> 使用DMA，最核心就是配置要传输的数据，包括数据从哪里来，要到哪里去，传输数据的单位是什么，要传多少数据，是一次传输还是循环传输等等。

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/b5197bc30ec740ee941661cfd0a79ee2.png" alt="img" style="zoom:50%;" />

1. 数据转运两大站点：存储器、外设寄存器。
   - DMA传输数据方向有三个：从外设到存储器，从存储器到外设，从存储器到存储器。

2. 外设、存储器所需数据：
   - 两者的起始地址：这两个参数决定数据从哪里来到哪里去；
   - 数据宽度：指定一次转运要按多大的数据宽度来进行，可选字节（uint8_t），半字（uint16_t），字（uint32_t）：
   - 地址是否自增：决定下次转运是不是要把地址移到下一个位置去，其参数可以选择使能或者失能：

3. 传输计数器：表示DMA需要转运几次。
   - 可将其理解为一个自减计数器，假如初始化值为5，那么每次转运一次计数减1，当减到0时DMA就不会再进行转运了，并且当其减到0，之前自增的地址又会回到起始地址，方便新一轮的转换。

4. 自动重装器：即传输模式选择：一次/循环传输
   - 当转运次数归零后，询问是否将转运次数回到最初值。如果配置为循环模式，DMA计数归零回到起始地址，而自动重装器又将DMA的数据恢复，这样就可以循环。

5. 触发机制，主要配置其使能或者失能，使能软件触发，失能硬件触发。
   - 注意：软件触发不能和循环一起使用。因为软件触发就是想将传输计数器清零，但是循环模式清零后会进行自动重装，因此不能一起使用。

## DMA请求-触发源选择

对于硬件触发需要根据不同的触发源，选择不同的通道，软件触发就随便了。

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image-20260813152757584.png" alt="image-20260813152757584" style="zoom: 33%;" />

如果外设要想通过DMA来传输数据，必须先给DMA控制器发送DMA请求，DMA收到请求信号之后，控制器会给外设一个应答信号， 当外设应答后且DMA控制器收到应答信号之后，就会启动DMA的传输，直到传输完毕。

DMA有DMA1和DMA2两个控制器，DMA1有7个通道，DMA2有5个通道，不同的DMA控制器的通道对应着不同的外设请求， 具体见DMA请求映像表。

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image-20260813095029414.png" alt="image-20260813095029414" style="zoom:40%;" />

### 仲裁器

当多个DMA请求同时到来时，意味着有先后响应处理的顺序问题，这个就由仲裁器管理。STM32的DMA仲裁器处理多请求时，其优先级判定分为两个明确的阶段：

- 软件优先级：有非常高、高、中和低4个等级，可在程序中配置。每个数据流或通道都可以被设置为四个等级之一。
- 硬件优先级：当两个或更多请求的软件优先级相同时，仲裁器会转而依靠硬件规则来裁决：通道编号越低优先权越高，比如通道0高于通道1。
- 在大容量产品和互联型产品中， DMA1控制器优先级高于DMA2控制器。

## DMA初始化结构体

HAL库函数对每个外设都建立了一个初始化结构体xxx_InitTypeDef(xxx为外设名称)，结构体成员用于设置外设工作参数，并由HAL库函数xxx_Init()调用这些设定参数进入设置外设相应的寄存器，达到配置外设工作环境的目的。

```C
//DMA_InitTypeDef初始化结构体
typedef struct {
    uint32_t Direction;            //传输方向，可选外设到存储器、存储器到外设以及存储器到存储器。
    uint32_t PeriphInc;            //使能外设地址自动递增，一般外设只有一个数据寄存器，故一般不会使能该位。
    uint32_t MemInc;               //使能存储器地址自动递增，自定义存储区一般是存放多个数据的，故一般要使能
    uint32_t PeriphDataAlignment;  //外设数据宽度，可选字节(8位)、半字(16位)和字(32位)
    uint32_t MemDataAlignment;     //存储器数据宽度
    uint32_t Mode;                 //DMA传输模式选择，可选一次传输或者循环传输，
    uint32_t Priority;             //数据流优先级，4个可选优先级：非常高、高、中和低
} DMA_InitTypeDef;
```

```c
#define DMA1_BASE             (AHBPERIPH_BASE + 0x00000000UL)
#define DMA1_Channel1_BASE    (AHBPERIPH_BASE + 0x00000008UL)
#define DMA1_Channel2_BASE    (AHBPERIPH_BASE + 0x0000001CUL)
#define DMA1_Channel3_BASE    (AHBPERIPH_BASE + 0x00000030UL)
#define DMA1_Channel4_BASE    (AHBPERIPH_BASE + 0x00000044UL)
#define DMA1_Channel5_BASE    (AHBPERIPH_BASE + 0x00000058UL)
#define DMA1_Channel6_BASE    (AHBPERIPH_BASE + 0x0000006CUL)
#define DMA1_Channel7_BASE    (AHBPERIPH_BASE + 0x00000080UL)

#define DMA2_BASE             (AHBPERIPH_BASE + 0x00000400UL)
#define DMA2_Channel1_BASE    (AHBPERIPH_BASE + 0x00000408UL)
#define DMA2_Channel2_BASE    (AHBPERIPH_BASE + 0x0000041CUL)
#define DMA2_Channel3_BASE    (AHBPERIPH_BASE + 0x00000430UL)
#define DMA2_Channel4_BASE    (AHBPERIPH_BASE + 0x00000444UL)
#define DMA2_Channel5_BASE    (AHBPERIPH_BASE + 0x00000458UL)


typedef struct __DMA_HandleTypeDef
{
  DMA_Stream_TypeDef         *Instance;        //DMA 通道/数据流的寄存器基地址，用于选择硬件资源（例如 DMA1_Stream0）
  DMA_InitTypeDef            Init;             //DMA 传输的核心配置参数
  HAL_LockTypeDef            Lock;             //用于实现资源锁，防止在中断和线程上下文中对同一个 DMA 句柄进行并发操作，保证线程安全。
  __IO HAL_DMA_StateTypeDef  State;            //当前 DMA 通道的状态，供 HAL 库内部进行状态管理
  void                       *Parent;          //指向拥有此DMA句柄的外设句柄，用于在回调函数中反查是哪个外设发起的传输。
    
  //回调函数指针：用于在传输完成、半完成、出错等事件发生时，回调用户定义的函数。
  void                       (* XferCpltCallback)( struct __DMA_HandleTypeDef * hdma); 
  void                       (* XferHalfCpltCallback)( struct __DMA_HandleTypeDef * hdma);
  void                       (* XferM1CpltCallback)( struct __DMA_HandleTypeDef * hdma);
  void                       (* XferM1HalfCpltCallback)( struct __DMA_HandleTypeDef * hdma);
  void                       (* XferErrorCallback)( struct __DMA_HandleTypeDef * hdma);
  void                       (* XferAbortCallback)( struct __DMA_HandleTypeDef * hdma);
  __IO uint32_t              ErrorCode;        //DMA 错误代码 
  DMA_TypeDef                *DmaBaseAddress;  //DMA 基地址
} DMA_HandleTypeDef;


//DMA状态
typedef enum
{
  HAL_OK       = 0x00U,
  HAL_ERROR    = 0x01U,
  HAL_BUSY     = 0x02U,
  HAL_TIMEOUT  = 0x03U
} HAL_StatusTypeDef;


// 完成DMA数据流参数配置
HAL_DMA_Init(DMA_HandleTypeDef *DMA_Handle);

// 启动DMA数据流传输
HAL_DMA_Start();
```

## 实验1：DMA存储器到存储器

存储器到存储器模式可以实现数据在两个内存的快速拷贝。

### 硬件设计

DMA存储器到存储器实验不需要其他硬件要求，可只用到RGB彩色灯用于指示程序状态。

### 软件设计

> 1. DMA数据流参数配置 DMA_Config()
>    - 使能DMA时钟；  __HAL_RCC_DMA1_CLK_ENABLE();
>    - 配置DMA数据参数；  HAL_DMA_Init();
>    - 使能DMA，进行传输；  HAL_DMA_Start();
> 2. 等待DMA传输完成；  __HAL_DMA_GET_FLAG(); //查询DMA传输过程中发生的各种事件状态，比如传输是否完成、是否出错等。
>
> 3. 对源数据和目标地址数据进行比较。
>
> *const关键字将数组变量定义为常量类型，数据存储在内部的FLASH中。*

## 实验2：DMA存储器到外设

### 软件设计

> 1. 配置USART/I2C/SPI通信功能；
>
> 2. 设置串口DMA工作参数；
>
>    - 使能DMA时钟；  __HAL_RCC_DMA1_CLK_ENABLE();
>
>    - 配置DMA数据参数；  HAL_DMA_Init();
>
>    - 将外设句柄和它要使用的DMA句柄“关联；__HAL_LINKDMA()   //链接DMA数据流及通道到串口外设通道上。
>
> 3. 使能DMA；
>
> 4. DMA传输同时CPU可以运行其他任务。
