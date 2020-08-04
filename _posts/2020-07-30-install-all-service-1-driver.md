---
layout: post
title:  "Android添加系统服务访问驱动程序系列:添加驱动程序"
date:   2020-07-30 00:00:00
catalog:  true
tags:
    - 系统
    - 驱动程序
    - Android

---



>  基于Android 6.0源码， 实操在Android系统中如何添加一个驱动程序。

## 1. 概述

开发一个驱动程序 mychar，mychar 是一个简单的字符设备驱动，Android 上层可以通过 read、write 对驱动进行读写操作，即写数据到 FIFO 和从 FIFO 读出写入的数据。 步骤如下：

1.  编写内核驱动程序模块
2.  修改内核 Kconfig 文件
3.  修改内核 Makefile文件
4.  修改内核 configs 文件

## 2. 编写内核驱动程序模块

注意：由于本文是基于高通平台的Android6.0的代码，因此有些文件的路径与AOSP有些出入。

驱动程序mychar的目录结构如下：

```
/kernel/drivers
---mychar
   ---mychar.c
   ---Kconfig
   ---Makefile
```

它由三个文件组成，其中 mychar.c 是源代码文件，Kconfig 是编译选项配置文件， Makefile 是编译脚本文件。下面我们就分别介绍这三个文件的实现。

### 2.1 驱动程序源代码

[->/kernel/drivers/mychar/mychar.c]

```c
#include <linux/module.h>
#include <linux/miscdevice.h>
#include <linux/fs.h>
#include <asm/uaccess.h>

#define BUF_SIZE 200

static char buf[BUF_SIZE];
static char *read_ptr;
static char *write_ptr;

static int device_open(struct inode *inode, struct file *file)
{
  printk("device_open called \n");

  return 0;
}

static int device_release(struct inode *inode, struct file *file)
{
  printk("device_release called \n");

  return 0;
}

static ssize_t device_read(struct file *filp,   /* see include/linux/fs.h   */
               char *buffer,    /* buffer to fill with data */
               size_t length,   /* length of the buffer     */
               loff_t * offset)
{
  int chars_read = 0;

  printk("device_read called \n");

  while(length && *read_ptr && (read_ptr != write_ptr)) {
    put_user(*(read_ptr++), buffer++);

    printk("Reading %c \n", *read_ptr);

    if(read_ptr >= buf + BUF_SIZE)
      read_ptr = buf;

    chars_read++;
    length--;
  }

  return chars_read;
}

static ssize_t
device_write(struct file *filp, const char *buff, size_t len, loff_t * off)
{
  int i;

  printk("device_write called \n");

  for(i = 0; i < len; i++) {
    get_user(*write_ptr, buff++);
    printk("Writing %c \n", *write_ptr);
    write_ptr++;
    if (write_ptr >= buf + BUF_SIZE)
      write_ptr = buf;
  }

  return len;
}

static struct file_operations fops = {
  .open = device_open,
  .release = device_release,
  .read = device_read,
  .write = device_write,
};

static struct miscdevice my_char_misc = {
  .minor = MISC_DYNAMIC_MINOR,
  .name = "mychar",
  .fops = &fops,
};

//驱动的入口，驱动从内核中装载时调用
//函数格式必须如 int xxx_init(void) 
int my_char_enter(void)
{
  int retval;

  retval = misc_register(&my_char_misc);
  printk("MY CHAR Driver got retval %d\n", retval);
  printk("mmap is %08X\n", (int) fops.mmap);

  read_ptr = buf;
  write_ptr = buf;

  return 0;
}

//驱动的出口，驱动从内核中卸载时调用(驱动被编译成模块时才需要卸载)
//函数格式必须如 void xxx_exit(void)
void my_char_exit(void)
{
  misc_deregister(&my_char_misc);
}

module_init(my_char_enter);// 内核装载时调用，my_char_enter通过module_init注册到系统中
module_exit(my_char_exit);// 内核卸载时调用

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Vane");
MODULE_DESCRIPTION("A driver Test Module");
```

这个文件向用户空间提供了 write 和 read 两个接口来访问虚拟硬件设备 mychar。mychar 注册为一个 misc 设备，其本质是一个字符设备。具体函数说明见代码注释。

### 2.2 驱动程序 Kconfig 文件

[->/kernel/drivers/mychar/Kconfig]

```makefile
menuconfig DRIVER_FOR_TEST
    bool "Drivers for test"
    help
      Drivers for test.
      If unsure, say no.
if DRIVER_FOR_TEST
config MY_CHAR_DRIVER
    tristate "mychar"
    help
      mychar driver.
endif
```

 Kconfig 的格式：

-  menuconfig 表示菜单（本身属于一个菜单中的项目，但是其又有子菜单项目）
- config 表示菜单中的一个配置项（本身并没有子菜单下的项目） 
-  menuconfig 或者 config 后面用空格隔开的大写字母，表示的就是这个配置项的配置项名字。这个字符串前面添加CONFIG_ 后就构成了 .config 中的配置项名字。 
-  一个 menuconfig 后面跟着的所有config项就是这个 menuconfig 的子菜单。
-  内核源码目录树中每一个 Kconfig 都会 source 引入其所有子目录下的 Kconfig，从而保证了所有的 Kconfig 项目都被包含进 nuconfig 中。 

tristate、bool 和 help 的含义：

- tristate 的意思就是这个配置项可以有三种选择
-  bool 的意思是这个配置项只能有2种选择
-  help 的意思是帮助信息，告诉我们这个配置项的含义，以及如何去配置它

### 2.3 驱动程序 Makefile 文件

```makefile
# Makefile for the mychar drivers.
obj-$(CONFIG_MY_CHAR_DRIVER)     += mychar.o

```

这是驱动程序的编译脚本，其中$(CONFIG_MY_CHAR_DRIVER)是一个变量，它的值与 Kconfig 的 config 变量有关，该值是 Kconfig 文件中 config 中的变量 MY_CHAR_DRIVER 加上前缀 CONFIG_而来的，必须要一致才行。

## 3. 修改内核 Kconfig 文件

[->/kernel/drivers/Kconfig]

```makefile
menu "Device Drivers"

source "drivers/base/Kconfig"
...
source "drivers/mychar/Kconfig"

endmenu

```

添加`source "drivers/mychar/Kconfig"`

## 4. 修改内核 Makefile 文件

[->/kernel/drivers/Makefile]

```makefile
...
obj-$(CONFIG_MY_CHAR_DRIVER) += mychar/
```

在末尾添加 `obj-$(CONFIG_MY_CHAR_DRIVER) += mychar/`

## 5. 修改内核 configs 文件

[->/kernel/arch/arm/configs/xxx_defconfig]

```makefile
...
CONFIG_DRIVER_FOR_TEST=y
CONFIG_MY_CHAR_DRIVER=y
```

上面文件xxx是根据所使用的芯片决定的，如果是使用64位的，则上面的arm改为 arm64。在 xxx_defconfig 文件末尾添加上述的配置变量为y，不同的值有不同的含义：

- 如果变量为y，意味着该驱动模块会被编进内核，上述的 mychar 也会编成 mychar.o。
- 如果变量为m，意味着该驱动模块单独编译成一个模块，上述的 mychar 编成 mychar.ko，内核可以单独装载和卸载该模块。
- 如果上述的 xxx_defconfig 文件没有模块的变量，该模块不编译

## 6. 赋予 /dev/mychar 权限

正常来说，内核创建的/dev/mychar节点的权限是crw------- root     root的，除了root 用户其他用户都不能访问，因此需要在 init 进程修改 /dev/mychar 的权限，使其他用户可以对 /dev/mychar 进行读写。

[->/system/core/rootdir/ueventd.rc]

```ini
...
/dev/mychar               0660   system     system
```

在文件末尾添加上面的代码，意味着 system用户可以对 /dev/mychar 进行读写。

## 7. 编译驱动程序并验证

由于是在Android源码上添加驱动模块，因此直接使用编译内核的方法编译新添加的驱动程序。`source`和`lunch`后，使用`make bootimage`命令编译内核。

将编译出来的 boot.img 镜像采用 fastboot 工具烧录到Android 设备上进行验证。如果有/dev/mychar 设备节点，说明驱动程序加载到内核中了。接着使用`adb shell "echo hello mychar > /dev/mychar"`命令往 mychar 设备节点写入hello mychar数据。接着使用`adb shell cat /dev/mychar`从 mychar 设备节点中读取刚才写入的数据，如果打印为 hello mychar ，则说明mychar 驱动正常工作。

```
C:\Users>adb shell "echo hello mychar > /dev/mychar"

C:\Users>adb shell cat /dev/mychar
hello mychar
```

大功告成！

