---
title: uC/OS-II在STM32F103上的移植
tags: 'uC/OS, stm32'
category: EmbededSystem
date: 2016-05-21 00:00:00
---


## uC/OS工程的创建和移植
先在[官方](https://www.micrium.com/download/micrium_stm32xxx_ucos-ii/)下载uc/os的源代码，下载链接如下，注册之后即可以下载(注意IAR和MDK的区别，IAR版汇编的在MDK上汇编不兼容，改动会比较多)

然后在Keil中新建一个uCOS的工程，选择板子为STM32F103C8，选择CMSIS下的CORE和Device下的Startup，以及Device下的StdPeriph Drivers下的Framework，RCC，和GPIO。新建完成后可以写一个简单的LED灯的Demo测试下灯和工程是否正常

![](/images/ucos-stm32/led-demo.png)

<!-- more -->

关于uCOS移植到STM32(Cortex-M3)具体可以参考官方的AN-1018文档。下图是uCOS移植到Cortex-M3的相关代码和模块之间的关系。其中最上面的是应用层，即是我们自己写的程序，然后下面是uCOS源代码，主要对应的是uCOS-II中的Src和Ports文件夹里面的代码。在工程目录和实际目录中新建Output, User和uCOS-II文件夹，分别对应上面的几个模块，其中Output用来作为工程的输出，在其中建立两个List和Obj两个文件夹，User下用户程序，User下图的BSP模块为官方的板级支持文件，这里不需要，uCOS-II下建立两个目录Src和Ports目录，分别和源代码对应。

![](/images/ucos-stm32/relationship.png)

然后在工程的Options的Output下，Select Foldeer for Objects，选择 Output目录下的Obj文件夹，并且同时勾选Create HEX File，Listing目录下Select Folder for Listings选择Output目录下的List文件夹。

![](/images/ucos-stm32/proj_struct.png)

在工程上右击Manage Project Items，添加几个和工程目录中相同的Group。

将Micrium\Software\uCOS-II\Ports\ARM-Cortex-M3\Generic\RealView这个文件夹中的内容拷贝到uCOS-II\uCOS-Port目录, uCOS-II\Source这个文件夹中的内容拷贝到uCOS-II \uCOS-Src目录，然后分别将其加入工程中的对应的Group中。为了uCOS的源代码被误改，可以将uCOS的Src目录下的文件设置成只读。

将Micrium\Software\EvalBoards\ST\STM3210B-EVAL\RVMDK\OS-Probe目录下的app_cfg.h，os_cfg.h和includes.h文件拷贝到User目录下，加入对应Group中，并且在User中添加入一个应用程序文件app.c。最终代码结构可见上图。

为了所有的能够直接include，将图中以下的目录添加到Options的C/C++标签下面的Include Paths中，并在Define中加上USE_STDPERIPH_DRIVER。

![](/images/ucos-stm32/preprocessor_sym.png)

修改os_cfg.h文件，#define OS_APP_HOOKS_EN 1为0，禁用钩子函数。

编译之后会提示os_cpu_c.c文件中OS_CPU_SysTickClkFreq函数没有定义，这里可以之间将其改正72MHZ（Cortex-M3），当然如果为了更好的适配性，可以将其改成system_stm32f10x.c文件下的一个变量SystemCoreClock。

```c
cnts = 72000000 / OS_TICKS_PER_SEC
```

重新编译，发现includes.h中很多文件不存在，直接将其注释掉即可。

```c
// #include <cpu.h>
// #include <lib_def.h>
// #include <lib_mem.h>
// #include <lib_str.h>

#include <stm32f10x_conf.h>
// #include <stm32f10x_lib.h>

#include <app_cfg.h>
// #include <lcd.h>
// #include <bsp.h>

// #if (APP_OS_PROBE_EN == DEF_ENABLED)
// #include <os_probe.h>
// #endif

// #if (APP_PROBE_COM_EN == DEF_ENABLED)
// #include <probe_com.h>

// #if (PROBE_COM_METHOD_RS232 == DEF_ENABLED)
// #include <probe_rs232.h>
```

上面只是一些简单的文件的复制，而要将ucos移植到stm32还需要使他们建立联系。uCOS-II的核心就是任务调度，其必须使用STM32的一个PendSV这个中断，其实这个就是小作业中做过的Cortex-M3的uCOS的任务切换函数。还有就是uCOS-II需要一个基准时间，其使用STM32中的一个专为操作系统设置的定时器SysTick。比较简单的解决方案是，在启动文件startup_stm32f10x_md.s中将所有的PendSV_Handler全部替换成为OS_CPU_PendSVHandler, SysTick_Handler 替换成OS_CPU_SysTickHandler，每个各有三处。也可以采用网上说的在外面实现这两个函数。

这样就完成了uCOS到STM32的整个移植过程。

## uC/OS对GPIO的访问
对于GPIO的访问，这里采用了点亮LED灯的方式进行实现。

在app.c中写入一些的代码，其新建了两个任务，分别对应用来点亮两个LED灯。系统首先进行初始化，然后创建两个任务。

```c
#include "includes.h"
#include "ucos_ii.h"
#define LED0STKSIZE 1000
#define LED1STKSIZE 1000

void LED_GPIO_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStruct;
    RCC_DeInit();
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
    GPIO_InitStruct.GPIO_Pin = GPIO_Pin_9 | GPIO_Pin_10;    
    GPIO_InitStruct.GPIO_Mode = GPIO_Mode_Out_PP;
    GPIO_InitStruct.GPIO_Speed = GPIO_Speed_2MHz;
    GPIO_Init(GPIOA, &GPIO_InitStruct);
}    

void Task_LED0(void *p_arg)
{
    while(1)
    {
        GPIO_SetBits(GPIOA, GPIO_Pin_9);    //on
        OSTimeDly(50);                    //half second
        GPIO_ResetBits(GPIOA, GPIO_Pin_9);    //off
        OSTimeDly(50);
    }
}

void Task_LED1(void *p_arg)
{
    while(1)
    {
        GPIO_SetBits(GPIOA, GPIO_Pin_10);    //on
        OSTimeDly(100);    //one second
        GPIO_ResetBits(GPIOA, GPIO_Pin_10);    //off
        OSTimeDly(100);
    }
}

OS_STK LED0TaskStk[LED0STKSIZE];
OS_STK LED1TaskStk[LED1STKSIZE];
int main()
{
    LED_GPIO_Init();    //initialize the gpio
    OSInit();                    //initialize the os
    OS_CPU_SysTickInit();    //initialze the system clock
    //create two task LED0 and LED1
    OSTaskCreate(Task_LED0, (void *)0, &LED0TaskStk[LED0STKSIZE - 1], 5);
    OSTaskCreate(Task_LED1, (void *)0, &LED1TaskStk[LED1STKSIZE - 1], 6);
    OSStart();    //start the os
    return 0;
}
```

将LED等连接到好之后，下载观察到两盏小灯闪烁

![](/images/ucos-stm32/led.png)

## 七段数码管的显示

![](/images/ucos-stm32/7seg-led-pin.png)

LG3641BH的引脚图如上面两幅图所示，其中A-G分别对应一个数字的七段(A-G对应如右图所示)，DP表示小数点，D1-D4表示位选信号。

每次我们只需要通过位选信号来显示一个数字即可以，然后利用时分复用的原理依次快速点亮其他的数字，使得人眼看起来是同时亮起来的。

实验中采用了GPIO的PA0-7引脚作为七段数码端的7段以及DP，PA9-12作为四个位选信号D1-D4。

下面为七段数码管的显示代码，digitSel为位选设置，通过传入的一个位选信号，设置PA9-12这几个位选信号的值。digitDisp为显示信号，根据传入的一个 数字以及小数点然后设置PA0-7这几个GPIO的值。

TaskDisp为显示七段数码管的任务，其中不断的扫描每一位数字，使得人眼看上去是一起显示的。这里需要注意的是由于位选控制和数字显示有前后之分，因此会出现某一个变化了另一个还没有变化的情况，因此这里将时间比较长的数字显示放在前面以此缩短两个GIPO的不统一的时间。而且后面加上了一个循环7000的延时，如果采用uCOS自带的定时函数，由于其精度太低，会出现闪烁，而如果这个计数设置的过小，则会出现重影。

TaskInc为value的递增任务，没个1秒其增加1，而且其任务的优先级需要比TaskDisp高，否则其不会得到执行。

```c
#include "includes.h"
#include "ucos_ii.h"

#define LED0STKSIZE 1000
#define LED1STKSIZE 1000

void GPIO_Config(void)
{
    GPIO_InitTypeDef GPIO_InitStruct;
    RCC_DeInit();
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);

    //digit select
    GPIO_InitStruct.GPIO_Pin = GPIO_Pin_9 | GPIO_Pin_10 | GPIO_Pin_11 | GPIO_Pin_12;
    GPIO_InitStruct.GPIO_Mode = GPIO_Mode_Out_PP;
    GPIO_InitStruct.GPIO_Speed = GPIO_Speed_2MHz;
    GPIO_Init(GPIOA, &GPIO_InitStruct);

    //7-segment and one point
    GPIO_InitStruct.GPIO_Pin = GPIO_Pin_0 |
                                                         GPIO_Pin_1 |
                                                       GPIO_Pin_2 |
                                                         GPIO_Pin_3 |
                                                         GPIO_Pin_4 |
                                                       GPIO_Pin_5 |
                                                       GPIO_Pin_6 |
                                                       GPIO_Pin_7 ;

    GPIO_Init(GPIOA, &GPIO_InitStruct);
}

void digitSel(int sel)
{
        uint16_t pinSel[] = {GPIO_Pin_9, GPIO_Pin_10, GPIO_Pin_11, GPIO_Pin_12};
        int i = 0;
        for(;i<4;++i)
        {
            if(sel == i)
                GPIO_SetBits(GPIOA, pinSel[i]);
            else
                GPIO_ResetBits(GPIOA, pinSel[i]);
        }
}

void digitDisp(int digit, int point)
{
    int node, i;
    uint16_t pinSel[] = {GPIO_Pin_0, GPIO_Pin_1, GPIO_Pin_2, GPIO_Pin_3,GPIO_Pin_4, GPIO_Pin_5, GPIO_Pin_6, GPIO_Pin_7};
    int sel = 1<< 7;

    switch (digit)
    {
        case 0 : node = 0xFC; break; // 11111100
    case 1 : node = 0x60; break; // 01100000
    case 2 : node = 0xDA; break; // 11011010
    case 3 : node = 0xF2; break; // 11110010
    case 4 : node = 0x66; break; // 01100110
    case 5 : node = 0xB6; break; // 10110110
    case 6 : node = 0xBE; break; // 10111110
    case 7 : node = 0xE0; break; // 11100000
    case 8 : node = 0xFE; break; // 11111110
    case 9 : node = 0xF6; break; // 11110110
    default: node = 0x9E; break; // 10011110 error and display E to represent error
   }
     node |= (point != 0);

     for(i=0;i<8;i++)
     {
        if((node & sel) == 0)
            GPIO_SetBits(GPIOA, pinSel[i]);
        else
            GPIO_ResetBits(GPIOA, pinSel[i]);

        sel >>= 1;
     }
}

int value = 0;
void TaskDisp(void *p_arg)
{
    int i, base, cnt = 7000;
    while(1)
    {
        base = 1000;
        for(i=0;i<4;i++)
        {
            digitDisp((value/base)%10, 0);
            digitSel(i);
            base /= 10;
            while(cnt--);
            cnt = 8000;
        }
    }
}

void TaskInc(void *p_arg)
{
    while(1)
    {
        value++;
        value %= 10000;
        OSTimeDly(100);
    }
}

OS_STK TaskStk1[LED0STKSIZE];
OS_STK TaskStk2[LED1STKSIZE];

int main()
{
    GPIO_Config();    //initialize the gpio

    OSInit();                    //initialize the os
    OS_CPU_SysTickInit();    //initialze the system clock

    //create two task LED0 and LED
    OSTaskCreate(TaskDisp, (void *)0, &TaskStk1[LED0STKSIZE - 1], 5);
    OSTaskCreate(TaskInc, (void *)0, &TaskStk2[LED0STKSIZE - 1], 4);
    OSStart();    //start the os

    return 0;
}
```

运行之后download观看显示结果（显示的测试显示的1234的结果）

![](/images/ucos-stm32/7seg-display.jpg)
