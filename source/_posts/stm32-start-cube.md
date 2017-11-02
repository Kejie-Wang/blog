---
title: STM32上电启动以及Cube库的程序编写
tags: 'stm32, cube, keil'
category: EmbededSystem
date: 2016-04-17 11:29:00
---


# 器材
## 硬件
- STM32F103核心板板一块
- microUSB线一根(供电)
- STLink板或USB串口板一块

## 软件

- 交叉编译软件。
- STM32CubeMX, Keil
- 串口软件Putty
<!-- more -->
#  连接示意图

![](/images/stm32-start-cube/conn.png)

# 实验步骤

## 串口输出Hello

### STM32CubeMX

首先安装STM32CubeMX，并且下载相应的STM32的库，由于实验中采用的是STM32F103C8，因此这里下载STM32F1的库，如下图所示![](/images/stm32-start-cube/stm32-lib.png)

然后新建一个工程，选择正确的板子型号（STM32F103C8Tx），定义引脚，这里定义PA.09为USART1_TX，PA.10为USART1_RX，并且设定模式为异步，Hardware Flow Control为Disable![](/images/stm32-start-cube/pin-def.png)

### Keil

再安装Keil软件，用来生成hex文件并且用其通过ST-LINK来下载到板子上运行。初次安装的时候需要下载相应的STM32设备包。

打开main.c中可以看到串口的初始化函数，按照要求设定串口的波特率为9600，数据的宽度为8，停止位为一位，没有检验位，然后用HAL_UART_Init函数初始化。

```c
/* USART1 init function */
void MX_USART1_UART_Init(void)
{
  huart1.Instance = USART1;
  huart1.Init.BaudRate = 9600;
  huart1.Init.WordLength = UART_WORDLENGTH_8B;
  huart1.Init.StopBits = UART_STOPBITS_1;
  huart1.Init.Parity = UART_PARITY_NONE;
  huart1.Init.Mode = UART_MODE_TX_RX;
  huart1.Init.HwFlowCtl = UART_HWCONTROL_NONE;
  huart1.Init.OverSampling = UART_OVERSAMPLING_16;
  HAL_UART_Init(&huart1);
}
```

这样子即可以采用HAL_UART_Transmit和HAL_UART_Receive两个函数进行串口的收发了。但是为了简单起见，这里重写了printf函数，以便可以用printf进行写入串口。重写int __io_putchar(int ch)和int fputc(int ch, FILE *f)两个函数

```c
#ifdef __GNUC__
#define PUTCHAR_PROTOTYPE int __io_putchar(int ch)
#else
#define PUTCHAR_PROTOTYPE int fputc(int ch, FILE *f)
#endif /* __GNUC__ */
PUTCHAR_PROTOTYPE
{

  HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, 0xFFFF);
  return ch;
}
```

这样即可以采用 printf 函数进行串口发送数据了，在 main 中加入一行printf(“Hello”);先串口写入 Hello 的函数。

### ST-LINK 下载

开始配置 Keil，打开设置后首先选定设备为 STM32F103C8，然后在Output中勾选Create HEX File选项。然后在解散ST-LINk之后可以在Debug选项中可以看到ST-LINK的调试器，并且打开设备可以看到相关的设备名字说明ST-LINK连接成功。在连接ST-LINK之前需要安装ST-LINK的驱动，然后在设备管理器中就可以看到已经成功挂载的设备。

在以上设置完毕以后，编译源代码，然后选择Flash-> download就可以下载了。

### Putty

后采用串口软件Putty读取串口上的数据，设置串口的属性如下:![](/images/stm32-start-cube/putty-setting.png)

打开串口，并且按下STM32上面的reset键就可以在串口上看到Hello字样



## 开关检测

在STM32CubeMX中加入两个GPIO输入PA11和PA12，设定其为GPIO_Input

然后在Keil中修改GPIO的初始化函数如下，设定PA11的方式为上拉输入，PA12为下降沿中断触发，初始化两个GPIO。

```c
void MX_GPIO_Init(void)
{
  GPIO_InitTypeDef GPIO_InitStruct;

  /* GPIO Ports Clock Enable */
  __HAL_RCC_GPIOA_CLK_ENABLE();

  /*Configure GPIO pins : PA11 PA12 */
  GPIO_InitStruct.Pin = GPIO_PIN_11;   
    GPIO_InitStruct.Mode = GPIO_MODE_INPUT;   
    GPIO_InitStruct.Pull = GPIO_PULLUP;   
    GPIO_InitStruct.Speed = GPIO_SPEED_LOW;   
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);   
    GPIO_InitStruct.Pin = GPIO_PIN_12;   
    GPIO_InitStruct.Mode = GPIO_MODE_IT_FALLING;   
    GPIO_InitStruct.Pull = GPIO_PULLUP;   
    GPIO_InitStruct.Speed = GPIO_SPEED_LOW;   
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
}
```

在main函数中加入GPIO的初始化函数，并且加入对于PA11的按下的检测函数，注意需要加上防抖动处理，否则会按下输出很多的Pressed

```c
while (1)
{
 /* USER CODE END WHILE */

 /* USER CODE BEGIN 3 */
        if(HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_11) == 0)
        {
                HAL_Delay(100);        //anti-jitter
                if(HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_11) == 0)
                {
                    printf("Pressed\r\n");
                }
        }
  }
```

重新build工程，然后下载到板子上通过串口查看结果



## PA12**中断响应**

在上面PA11的基础上进行修改，因此这里不需要添加引脚，不用STM32CubeMX。

首先配置PA12为下降边沿触发，在GPIO的初始化函数中如下设置

```c
GPIO_InitStruct.Pin = GPIO_PIN_12;   
GPIO_InitStruct.Mode = GPIO_MODE_IT_FALLING;   
GPIO_InitStruct.Pull = GPIO_PULLUP;   
GPIO_InitStruct.Speed = GPIO_SPEED_LOW;   
HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
```

设置中断的优先级，enable中断向量 表处理，在main函数中添加以下代码

```c
HAL_NVIC_SetPriority(EXTI15_10_IRQn, 0, 0);
HAL_NVIC_EnableIRQ(EXTI15_10_IRQn);
```

当出现中断的时候，由于设定的引脚为PA12，因此当出现中断的时候，STM32会调用EXTI15_10_IRQHandler这个中断服务程序进行处理，我们在stm32f1xx_it.c中实现它，并在里面调用HAL_GPIO_EXTI_IRQHandler函数。由于上述函数会判断中断标志位并且调用HAL_GPIO_EXTI_Callback函数，因此我们将中断处理程序放在了HAL_GPIO_EXTI_Callback函数中进行实现

```C
void EXTI15_10_IRQHandler(void)
{
    HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_12);
}

void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    flag = 1;
    counter = counter + 1;
}
```

对于全局变量的定义，我们在main.c中定影了两个全局变量flag和counter，然后在stm32f1xx_it.c中用extern关键词声明这两个变量以此达到两个c文件共享全局变量的效果。

然后在main的主循环中判断标志flag，如果其为1，则输出counter值，并且消除标志位。这里为了输出整形变量的值，我写了一个从unsigned int到字符串的函数，以便采用printf进行输出。

```c
while (1)
  {
  /* USER CODE END WHILE */
  /* USER CODE BEGIN 3 */
        if(HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_11) == 0)
        {
            HAL_Delay(100);        //anti-jitter
            if(HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_11) == 0)
            {
                    printf("Pressed\r\n");
            }
        }
        if(flag)
        {
            char s[33];
            toString(counter, s);
            printf("%s\r\n", s);
            flag = 0;
        }

  }

/**
*transform a integer into a string
*@num: The input number
*@s: The string after transforming
*@return void
*/
void toString(int num, char s[])
{
    int len = 0, tmp = num;
if(tmp==0)
    len=1;
    while(tmp != 0)
    {
        len++;
        tmp /= 10;
    }
    s[len--] = '\0';
    while(num != 0)
    {
        s[len--] = (num%10) + '0';
        num /= 10;
    }
}
```

重新编译代码，下板执行。通过下图可以看到，每次按下按钮会输出一个递增的数字，说明程序运行正确。



## 时钟中断

首先采用STM32CubeMX启用TIM3，并且采用内部时钟

然后生成代码，用Keil打开工程。

采用内部时钟，其频率为8MHz，配置时钟向上计数，计数到199，Prescaler的分频范围为0-65536，因此在这里设置为8000，，8MHz/8000 = 1000Hz = 1ms。

修改MX_TIM3_Init函数体如下，分别设置时钟为TIM3，Prescaler为8000，计数模式为向上计数，周期为199，时钟为内部时钟以及复位模式。然后设置中断的优先级，enable中断向量表处理。最后初始化定时器。

```c
/* TIM3 init function */
void MX_TIM3_Init(void)
{
  TIM_ClockConfigTypeDef sClockSourceConfig;
  TIM_MasterConfigTypeDef sMasterConfig;

    TIM_Handle.Instance = TIM3;   
    TIM_Handle.Init.Prescaler = 8000;   
    TIM_Handle.Init.CounterMode = TIM_COUNTERMODE_UP;   
    TIM_Handle.Init.Period = 199;

  sClockSourceConfig.ClockSource = TIM_CLOCKSOURCE_INTERNAL;
 // HAL_TIM_ConfigClockSource(&TIM_Handle, &sClockSourceConfig);

  sMasterConfig.MasterOutputTrigger = TIM_TRGO_RESET;
    HAL_NVIC_SetPriority(TIM3_IRQn, 0, 0);   
    HAL_NVIC_EnableIRQ(TIM3_IRQn);

  HAL_TIM_Base_Init(&TIM_Handle);
}
```

在stm32f1xx_hal_msp.c中的HAL_TIM_Base_MspInit中enable定时器时钟，在HAL_TIM_Base_MspDeInit中disable定时器时钟（如果采用STM32CubeMX生成，其会自动生成该代码，否则就需要自己添加）。

由于定时器中断调用TIM3_IRQHandler函数，因此我们在stm32f1xx_it.c中编写中断服务程序TIM3_IRQHandler函数。由于在其中要用HAL_TIM_IRQHandler

(&TIM_Handle),因此在stm32f1xx_it.c采用extern申明TIM_Handle。由于该服务程序会调用HAL_TIM_PeriodElapsedCallback回调函数，因此在其中加入中断处理的程序。在这里我加的是一个将标志位置位，然后将一个全局变量加上10的一个操作。

```c
void TIM3_IRQHandler(void)
{
    HAL_TIM_IRQHandler(&TIM_Handle);
}

void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    timeFlag = 1;
    counter += 10;
}
```

然后在main中HAL_TIM_Base_Start_IT(&TIM_Handle)启动定时器，在主循环中输出counter的值来检验定时中断。

```c
while (1)
{
/* USER CODE END WHILE */

/* USER CODE BEGIN 3 */
      //time interrupt
      if(timeFlag)
      {
          char s[33];
          toString(counter, s);
          printf("%s\r\n", s);
          timeFlag = 0;
      }
      /* USER CODE END 3 */
}
```

重新编译代码，将其载入板子运行，通过串口观看结果。通过下图可以看到，串口上不断输出了以10递增的一些列数字，说明定时中断正常工作。
