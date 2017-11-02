---
title: 树莓派上的GPIO字符驱动程序
tags: raspberry, gpio, 字符驱动程序
category: EmbededSystem
date: 2016-06-04 00:00:00
---

主要是在嵌入式Linux（树莓派）中如何使用已有的函数库编写应用程序操纵GPIO，如何编写字符设备驱动程序在内核程序中使用GPIO
<!-- more -->
# 硬件连接图
![树莓派和MAX7219连接图](/images/rasp-gpio/dev-conn.png)

# 虚拟文件系统操作GPIO
Linux可以通过访问sys/class/gpio下的一些文件，通过对这些文件的读写来实现对于GPIO的访问。
树莓派下面的可用的GPIO如下图所示(需要注意树莓派一代和二代的区别)：
![](/images/rasp-gpio/rasp-gpio-map.png)
首先用一个小灯来测试下操作。首先向export中写入18，表示启用18号gpio端口，执行之后，可以看到该目录下多出了一个gpio18的目录。进入该目录后，先direction中写入out，表示该gpio作为输出，然后向value文件中写入1，表示其输出1，小灯的一端接gpio18，另一端接地，就可以看到小灯点亮。实验成功。
![](/images/rasp-gpio/sys-eg.png)

将其改装成用C操作文件的方式进行，这里我写了一个简单的库，包括gpio的export和unexport以及read和write:
```c
//gpio_sys.h
#ifndef _GPIO_SYS_H_
#define _GPIO_SYS_H_

#include <stdio.h>
#include <stdlib.h>

#define BUFFMAX 3
#define SYSFS_GPIO_EXPORT     "/sys/class/gpio/export"
#define SYSFS_GPIO_UNEXPORT "/sys/class/gpio/unexport"
#define SYSFS_GPIO_DIR_IN    0
#define SYSFS_GPIO_DIR_OUT    1
#define SYSFS_GPIO_VAL_HIGH 1
#define SYSFS_GPIO_VAL_LOW    0

#define ERR(args...) fprintf(stderr, "%s\n", args);

int GPIOExport(int pin);
int GPIOUnexport(int pin);
int GPIODirection(int pin, int dir);
int GPIORead(int pin);
int GPIOWrite(int pin, int value);

#endif
```

```c
//gpio_sys.c
#include "gpio_sys.h"
#include <sys/stat.h>
#include <sys/types.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

int GPIOExport(int pin)
{
    char buff[BUFFMAX];

    int fd;
    if((fd=open(SYSFS_GPIO_EXPORT, O_WRONLY)) == -1)
    {
        ERR("Failed to open export for writing!\n");
        return -1;
    }

    int len = snprintf(buff, sizeof(buff), "%d", pin);
    if(write(fd, buff, len) == -1)
    {
        ERR("Failed to export gpio!\n");
        return -1;
    }

    if(close(fd) == -1)
    {
        ERR("Failed to close export!\n");
        return -1;
    }
    return 0;
}

int GPIOUnexport(int pin)
{
    char buff[BUFFMAX];
    int fd;
    if((fd=open(SYSFS_GPIO_UNEXPORT, O_WRONLY)) == -1)
    {
        ERR("Failed to open unexport for writing!\n");
        return -1;        
    }

    int len = snprintf(buff, sizeof(buff), "%d", pin);
    if(write(fd, buff, len) == -1)
    {
        ERR("Failed to unexport gpio!\n");
        return -1;
    }

    if(close(fd) == -1)
    {
        ERR("Failed to close unexport!\n");
        return -1;
    }
    return 0;
}

int GPIODirection(int pin, int dir)
{
    char dirCh[][5] =  {"in", "out"};
    char path[64];

    int fd;
    snprintf(path, sizeof(path), "/sys/class/gpio/gpio%d/direction", pin);
    printf(path);
    if((fd = open(path, O_WRONLY)) == -1)
    {
        ERR("Failed to open direction for writing!\n");
        return -1;    
    }

    if(write(fd, dirCh[dir], strlen(dirCh[dir])) == -1)
    {
        ERR("Failed to set direction!\n");
        return -1;        
    }

    if(close(fd) == -1)
    {
        ERR("Failed to close direction!\n");
        return -1;
    }
    return 0;
}

int GPIORead(int pin)
{
    char path[64];
    char buff[BUFFMAX];

    snprintf(path, sizeof(path), "/sys/class/gpio/gpio%d/value", pin);

    int fd;
    if((fd == open(path, O_RDONLY)) == -1)
    {
        ERR("Failed to open value for reading!\n");
        return -1;    
    }

    if(read(fd, buff, sizeof(buff)) == -1)
    {
        ERR("Failed to read value!\n");
        return -1;
    }

    if(close(fd) == -1)
    {
        ERR("Failed to close value!\n");
        return -1;
    }

    return atoi(buff);
}

int GPIOWrite(int pin, int value)
{
    char path[64];
    char valuestr[][2] = {"0", "1"};

    if(value != 0 && value != 1)
    {
        fprintf(stderr, "value = %d\n", value);
        ERR("Writing erro value!\n");
        return -1;
    }
    snprintf(path, sizeof(path), "/sys/class/gpio/gpio%d/value", pin);
    int fd;
    if((fd = open(path, O_WRONLY)) == -1)
    {
        ERR("Failed to open value for writing!\n");
        return -1;    
    }

    if(write(fd, valuestr[value], 1) == -1)
    {
        ERR("Failed to write value!\n");
        return -1;
    }

    if(close(fd) == -1)
    {
        ERR("Failed to close value!\n");
        return -1;
    }
    return 0;
}
```

![](/images/rasp-gpio/max7219.png)
然后编写MAX_7219的显示程序。

MAX_7219其输出引DIG0-7和SEG A-G连接了8*8的LED矩阵，而我们关心的则是其输入的5个引脚，分别是5V的VCC和接地GND，以及时钟信号CLK，片选信号CS，串行数据输入端口DIN
MAX_7219使用前需要对一些寄存器进行初始化设置，包括编码寄存器，亮度寄存器，模式寄存器以及显示检测寄存器。
![](/images/rasp-gpio/max7219-reg.png)
MAX_7219的数据的写入是在时钟上升沿写入的，连续数据的后16位被移入寄存器中，其中8-11为地址，0-7位为数据。
在了解了MAX_7219的工作原理之后就可以使用上面的gpio的操纵函数进行字母的显示了。下面给出的是在8*8的矩阵上依次显示0-9,a-z以及中国字样的程序。
采用的GPIO为gpio18，gpio23和gpio24分别连接CLK，CS以及DIN。
首先需要对于树莓派的GPIO口进行一些配置:
```c
void Init_MAX7219(void)    
{
     Write_Max7219(0x09, 0x00);       //encoding with BCD
     Write_Max7219(0x0a, 0x03);       //luminance
     Write_Max7219(0x0b, 0x07);       //scanning bound
     Write_Max7219(0x0c, 0x01);       //mode: normal
     Write_Max7219(0x0f, 0x00);       
}
```
初始化MAX_7219的一些寄存器，设置其译码方式为BCD译码，亮度为3，扫描界限为8个数码管显示，采用普通模式，正常显示
```c
void Init_MAX7219(void)
{
     Write_Max7219(0x09, 0x00);       //encoding with BCD
     Write_Max7219(0x0a, 0x03);       //luminance
     Write_Max7219(0x0b, 0x07);       //scanning bound
     Write_Max7219(0x0c, 0x01);       //mode: normal
     Write_Max7219(0x0f, 0x00);       
}
```
然后编写数据写入函数，其首先将地址写入，然后在将数据写入。其中写入的顺序为从高位到低位按位依次写入即可。写入的时候首先将CLK设为低电平，然后使得数据有限，然后将CLK设为高电平，让数据在上升沿时写入
```c
void Write_Max7219_byte(uchar DATA)         
{
    uchar i;   
    GPIOWrite(pinCS, SYSFS_GPIO_VAL_LOW);
    for(i=8;i>=1;i--)
    {
        GPIOWrite(pinCLK, SYSFS_GPIO_VAL_LOW);
        GPIOWrite(pinDIN, (DATA&0x80) >> 7);
        DATA <<= 1;
        GPIOWrite(pinCLK, SYSFS_GPIO_VAL_HIGH);
    }                                 
}

void Write_Max7219(uchar address,uchar dat)
{
     GPIOWrite(pinCS, SYSFS_GPIO_VAL_LOW);
     Write_Max7219_byte(address);           //writing address
     Write_Max7219_byte(dat);               //writing data
     GPIOWrite(pinCS, SYSFS_GPIO_VAL_HIGH);                      
}

void Init_MAX7219(void)
{
     Write_Max7219(0x09, 0x00);       //encoding with BCD
     Write_Max7219(0x0a, 0x03);       //luminance
     Write_Max7219(0x0b, 0x07);       //scanning bound
     Write_Max7219(0x0c, 0x01);       //mode: normal
     Write_Max7219(0x0f, 0x00);       
}
```
最后将上述过程拼接起来就可以了。
连接线路然后运行就可以看到字符输出了，下面列出了8， W，中的显示图像。
<img src="/images/rasp-gpio/ch-zhong-img.png" width="50%" height="50%">

```c
#define uchar unsigned char
#define uint  unsigned int

#define pinCLK     18
#define pinCS     23
#define pinDIN     24

uchar codeDisp[38][8]={
    {0x3C,0x42,0x42,0x42,0x42,0x42,0x42,0x3C},//0
    {0x10,0x18,0x14,0x10,0x10,0x10,0x10,0x10},//1
    {0x7E,0x2,0x2,0x7E,0x40,0x40,0x40,0x7E},//2
    {0x3E,0x2,0x2,0x3E,0x2,0x2,0x3E,0x0},//3
    {0x8,0x18,0x28,0x48,0xFE,0x8,0x8,0x8},//4
    {0x3C,0x20,0x20,0x3C,0x4,0x4,0x3C,0x0},//5
    {0x3C,0x20,0x20,0x3C,0x24,0x24,0x3C,0x0},//6
    {0x3E,0x22,0x4,0x8,0x8,0x8,0x8,0x8},//7
    {0x0,0x3E,0x22,0x22,0x3E,0x22,0x22,0x3E},//8
    {0x3E,0x22,0x22,0x3E,0x2,0x2,0x2,0x3E},//9
    {0x8,0x14,0x22,0x3E,0x22,0x22,0x22,0x22},//A
    {0x3C,0x22,0x22,0x3E,0x22,0x22,0x3C,0x0},//B
    {0x3C,0x40,0x40,0x40,0x40,0x40,0x3C,0x0},//C
    {0x7C,0x42,0x42,0x42,0x42,0x42,0x7C,0x0},//D
    {0x7C,0x40,0x40,0x7C,0x40,0x40,0x40,0x7C},//E
    {0x7C,0x40,0x40,0x7C,0x40,0x40,0x40,0x40},//F
    {0x3C,0x40,0x40,0x40,0x40,0x44,0x44,0x3C},//G
    {0x44,0x44,0x44,0x7C,0x44,0x44,0x44,0x44},//H
    {0x7C,0x10,0x10,0x10,0x10,0x10,0x10,0x7C},//I
    {0x3C,0x8,0x8,0x8,0x8,0x8,0x48,0x30},//J
    {0x0,0x24,0x28,0x30,0x20,0x30,0x28,0x24},//K
    {0x40,0x40,0x40,0x40,0x40,0x40,0x40,0x7C},//L
    {0x81,0xC3,0xA5,0x99,0x81,0x81,0x81,0x81},//M
    {0x0,0x42,0x62,0x52,0x4A,0x46,0x42,0x0},//N
    {0x3C,0x42,0x42,0x42,0x42,0x42,0x42,0x3C},//O
    {0x3C,0x22,0x22,0x22,0x3C,0x20,0x20,0x20},//P
    {0x1C,0x22,0x22,0x22,0x22,0x26,0x22,0x1D},//Q
    {0x3C,0x22,0x22,0x22,0x3C,0x24,0x22,0x21},//R
    {0x0,0x1E,0x20,0x20,0x3E,0x2,0x2,0x3C},//S
    {0x0,0x3E,0x8,0x8,0x8,0x8,0x8,0x8},//T
    {0x42,0x42,0x42,0x42,0x42,0x42,0x22,0x1C},//U
    {0x42,0x42,0x42,0x42,0x42,0x42,0x24,0x18},//V
    {0x0,0x49,0x49,0x49,0x49,0x2A,0x1C,0x0},//W
    {0x0,0x41,0x22,0x14,0x8,0x14,0x22,0x41},//X
    {0x41,0x22,0x14,0x8,0x8,0x8,0x8,0x8},//Y
    {0x0,0x7F,0x2,0x4,0x8,0x10,0x20,0x7F},//Z
    {0x8,0x7F,0x49,0x49,0x7F,0x8,0x8,0x8},//中
    {0xFE,0xBA,0x92,0xBA,0x92,0x9A,0xBA,0xFE},//国
};

void Delay_xms(uint x)
{
    uint i,j;
    for(i=0;i<x;i++)
        for(j=0;j<50000;j++);
}
int main(void)
{
    uchar i,j;
     Delay_xms(50);
     if(GPIOConfig() == -1)
     {
         ERR("Can not configure the gpio!\n")
         return 0;
     }
     Init_MAX7219();
     while(1)
     {
          for(j=0;j<38;j++)
          {
               for(i=1;i<9;i++)
                Write_Max7219(i,codeDisp[j][i-1]);
               Delay_xms(1000);
          }  
     }

     if(GPIORelease() == -1)
     {
         ERR("Release the gpio error!\n")
         return 0;
     }
}
```

# 字符驱动程序
对于通过寄存器操作GPIO，我们就需要知道树莓派上各个GPIO端口的寄存器的地址，在树莓派的CPU[bcm2825](https://www.raspberrypi.org/wp-content/uploads/2012/02/BCM2835-ARM-Peripherals.pdf)的芯片手册上可以查到
![](/images/rasp-gpio/bcm2825.png)
且由于树莓派的IO的空间的起始地址是0xF2000000，并且GPIO的偏移地址为0x200000，那么真实的GPIO的开始地址为0xF2200000,这个数据可以通过<mach/platform.h>这个头文件中的GPIO_BASE获得。
根据上面说列出的信息，就可以编写我们的对于GPIO端口的操作函数了，这里定义了对于GPIO口的function selection, set以及clear三个函数。
在网上的一篇博客中还看到可以通过树莓派提供的一些列的gpiochip的包装的函数进行操作，但是尝试了很久没有成功，驱动程序报错。编写这个程序也可以参考网上可以下载到的bcm2835的对于GPIO操作的一个bcm2835的一个开源的函数库

```c
//0->input  1-<output
static void bcm2835_gpio_fsel(int pin, int functionCode)
{
    int registerIndex = pin / 10;
    int bit = (pin % 10) * 3;

    unsigned oldValue = s_pGpioRegisters-> GPFSEL[registerIndex];
    unsigned mask = 0b111 << bit;
    printk("Changing function of GPIO%d from %x to %x\n",
           pin,
           (oldValue >> bit) & 0b111,
           functionCode);

    s_pGpioRegisters-> GPFSEL[registerIndex] =
        (oldValue & ~mask) | ((functionCode << bit) & mask);
}

static void bcm2835_gpio_set(int pin)
{
    printk("GPIO set %d\n oldValue=%d", pin, s_pGpioRegisters->GPSET[0]);
    s_pGpioRegisters-> GPSET[pin / 32] = (1 << (pin % 32));      
    printk("GPIO set %d\n oldValue=%d", pin, s_pGpioRegisters->GPSET[0]);
}

static void bcm2835_gpio_clr(int pin)
{
    printk("GPIO clear %d\n oldValue=%d", pin, s_pGpioRegisters->GPCLR[0]);
    s_pGpioRegisters-> GPCLR[pin / 32] = (1 << (pin % 32));
    printk("GPIO clear %d\n newValue=%d", pin, s_pGpioRegisters->GPCLR[0]);
}
```
在完成对于GPIO的操作函数之后就可以开始编写字符驱动程序了，Linux的字符驱动程序主要以模块的形式装载到内核中，然后应用程序通过文件的操作的方式对于硬件进行操作。编写一个Linux的字符驱动程序主要如下图所示：
![](/images/rasp-gpio/linux-ch-driver.jpg)
如上图所示，首先需要做的是对于驱动进行初始化设置，在模块的初始化函数中编写如下，首先获得一个设备号（静态或者动态获取），然后进行字符设备的初始化，主要是将file_operation这个结构体中的函数和我们的定义的函数进行一个绑定，注册字符设备，最后创建得到设备。然后在进行设备的初始化，这里首先获取的是GPIO寄存器的基地址，然后通过fsel函数设置相关的三个GPIO口的模式为OUT，然后通过这三个GPIO去初始化MAX7219（同上）
```c
static struct file_operations MAX7219_cdev_fops = {
    .owner = THIS_MODULE,
    .open = MAX7219_open,
    .write = MAX7219_write,
    .release = MAX7219_release,
};

static int MAX7219_init(void)
{
    int ret;

    MAX7219_dev_id = MKDEV(major, 0);
    if(major)    //static allocate
        ret = register_chrdev_region(MAX7219_dev_id, 1, DRIVER_NAME);
    else    //dynamic allocate
    {
        ret = alloc_chrdev_region(&MAX7219_dev_id, 0, 1, DRIVER_NAME);
        major = MAJOR(MAX7219_dev_id);
    }

    if(ret < 0)
        return ret;

    cdev_init(&MAX7219_cdev, &MAX7219_cdev_fops);    //initialize character dev
    cdev_add(&MAX7219_cdev, MAX7219_dev_id, 1);                //register character device
    MAX7219_class = class_create(THIS_MODULE, DRIVER_NAME);    //create a class
    device_create(MAX7219_class, NULL, MAX7219_dev_id, NULL, DRIVER_NAME);    //create a dev

    s_pGpioRegisters = (struct GpioRegisters*)__io_address(GPIO_BASE);

    printk("address = %x\n", (int)__io_address(GPIO_BASE));

    //gpio configure
    bcm2835_gpio_fsel(pinCLK, 1);
    bcm2835_gpio_fsel(pinCS, 1);
    bcm2835_gpio_fsel(pinDIN, 1);

    //initialize the MAX7219
    Init_MAX7219();

    printk("MAX7219 init successfully");
    return 0;
}
```
当这个内核模块退出的时候，需要通过device_destroy函数将这个设备删除，以便设备号可以提供给其他的设备宋，并且将其从注册中删除
```c
void MAX7219_exit(void)
{
    device_destroy(MAX7219_class, MAX7219_dev_id);
    class_destroy(MAX7219_class);
    unregister_chrdev_region(MAX7219_dev_id, 1);
    printk("MAX7219 exit successfully\n");
}
```

然后就需要编写file_operations中的函数，这里主要定义了用的open，write和release（close）函数，对于read，iocnl等等就没有进行定义。

Open函数比较简单，主要就是判断下文件是否已经打开，如果已经被打开了，那么其将不能被再次打开

```c
static int MAX7219_open(struct inode *inode, struct file *flip)
{
    printk("Open the MAX7219 device!\n");
    if(state != 0)
    {
        printk("The file is opened!\n");    
        return -1;
    }
    state++;
    printk("Open MAX7219 successfully!\n");
    return 0;
}
```
Release函数和Open相反，将打开的文件关闭即可
```c
static int MAX7219_release(struct inode *inode, struct file *flip)
{
    printk("Close the MAX7219 device!\n");
    if(state == 1)
    {
        state = 0;
        printk("Close the file successfully!\n");
        return 0;
    }
    else
    {
        printk("The file has closed!\n");
        return -1;
    }
}
```

 这里主要是write函数，其将用户送来的一个字符需要显示在MAX7219上，其显示的方法和前面的虚拟文件操作基本相同，只是GPIO口操作调用的函数不同。其首先需要将用户空间传递过来的参数通过copy_from_user函数拷贝到内核空间上，然后将调用相关的显示函数进行显示即可。
 对于MAX7219的显示函数这里就没有详细叙述，整个过程都和虚拟文件操作一样，只要将上面的接口改成寄存器操作的接口即可。

```c
static ssize_t MAX7219_write(struct file *filp, const char __user *buf, size_t len, loff_t *off)
{
    printk("Write %s into the MAX7219\n", buf);
    int ret, i;
    char ch;
    if(len == 0)
        return 0;

    if(copy_from_user(&ch, (void *)buf, 1))
        ret = -EFAULT;
    else
    {
        int index;
        if(ch >= '0' && ch <= '9')
            index = ch - '0';
        else if(ch >= 'A' && ch <= 'Z')
            index = ch - 'A' + 10;
        else if(ch >= 'a' && ch <= 'z')
            index = ch - 'a' + 10;
        else
            index = 36;    //unknown display 中
        printk("Write character %c, index=%d\n", ch, index);
        for(i=0;i<8;i++)
            Write_Max7219(i+1, codeDisp[index][i]);

        ret = 1;    //write a character
    }

    return ret;
}
```
编写完最后，编写Makefile文件进行编译，这个和上次实验内容相同，这里不再赘述。需要注意的是内核的模块版本号必须要和编译的相同，否则会出现ukonwn parameters之类的错误。上次做好久把内核删掉的只能重新编译一次内核了。
```shell
ARCH        := arm
CROSS_COMPILE    := arm-linux-gnueabi-

CC := $(CROSS_COMPILE)gcc
LD := $(CROSS_COMPILE)ld

obj-m := gpio_chdev.o

KERNELDIR := /home/jack/Documents/course/EmbededSystem/RaspberrySource/modules/lib/modules/4.4.11/build
PWD = $(shell pwd)

all:
    make -C $(KERNELDIR) M=$(PWD) modules
clean:
    rm -f *.o *.mod.c *.symvers *.order
```

编写如下的测试程序，以此显示A-Z以及0-9在MAX7219上面

```c
#include <stdio.h>  
#include <stdlib.h>  
#include <unistd.h>  
#include <sys/ioctl.h>  
#include <sys/time.h>
#include <sys/fcntl.h>
#include <sys/stat.h>  

void Delay_xms(uint x)
{
    uint i,j;
    for(i=0;i<x;i++)
        for(j=0;j<50000;j++);
}

int main(int argc, char **argv)  
{  
    int fd;
    int ret;

    fd = open("/dev/MAX7219", O_WRONLY);  
    if (fd < 0)
    {  
        fprintf(stderr, "Fail to open /dev/MAX7219!\n");
        exit(1);
    }
    char buff[1];
    int i=0;
    for(i=0;i<26;i++)
    {
        buff[0] = 'a' + i;
        if((ret = write(fd, buff, 1))<0)
        {
            fprintf(stderr, "Fail to write /dev/MAX7219! %d\n", ret);
            break;
        }
        Delay_xms(1000);
    }
    for(i=0;i<10;i++)
    {
        buff[0] = '0' + i;
        if((ret = write(fd, buff, 1))<0)
        {
            fprintf(stderr, "Fail to write /dev/MAX7219! %d\n", ret);
            break;
        }
        Delay_xms(1000);
    }
    fprintf(stdout, "Write /dev/MAX7219 successfully! %d\n", ret);
    close(fd);  
    return 0;  
}
```

# 参考链接

[基于树莓派的字符设备内核驱动程序框架编写](http://blog.csdn.net/embbnux/article/details/17712547?utm_source=tuicool&utm_medium=referral)

[树莓派linux启动学习之LED控制](http://blog.csdn.net/hcx25909/article/details/16860725)

[Creating a Basic LED Driver for Raspberry Pi](http://sysprogs.com/VisualKernel/tutorials/raspberry/leddriver/)

[BCM2835-ARM-Peripherals.pdf](https://www.raspberrypi.org/wp-content/uploads/2012/02/BCM2835-ARM-Peripherals.pdf)
