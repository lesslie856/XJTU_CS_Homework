# Exp3: 动态模块和字符设备驱动

## 实验目的

* 理解LINUX字符设备驱动程序的基本原理；
* 掌握字符设备的驱动运作机制；
* 学会编写字符设备驱动程序
* 学会编写用户程序通过对字符设备的读写完成不同用户的通信

## 实验内容

* 编写一个简单的字符设备驱动程序，以内核空间模拟字符设备，完成对该设备的打开，读写和释放操作；
* 编写聊天程序实现不同用户通过该设备实现一对一、一对多、多对多的聊天。



## 预备：动态模块

### 模块的操作

这里采用 `V 2.6` ，执行 `makefile` 进行编译

```
V2.6 当前目录建立Makefile文件 执行make即可按照Makefile的规定进行编译，形成.ko模块文件

模块的加载 insmod命令 如： insmod filename.ko 或insmod filename.o

模块的查看 lsmod more /proc/modules dmesg ——查看日志（printk）

模块的卸载 rmmod命令 如：rmmod filename

```

### 简单的hello world示例

`mymodules.c`文件

```
#include <linux/init.h>                                            /*必须要包含的头文件*/ 
#include <linux/kernel.h> 
#include <linux/module.h>                                    /*必须要包含的头文件*/ 
static int mymodule_init(void)                                //模块初始化函数 
{   
    printk("hello,my module wored! \n");
    /*输出信息到内核日志*/   
    return 0; 
} 
static void mymodule_exit(void) //模块清理函数 
{    
    printk("goodbye,unloading my module.\n");         
    /*输出信息到内核日志*/ 
}   
module_init(mymodule_init);                                //注册初始化函数 
module_exit(mymodule_exit);                              
//注册清理函数 
MODULE_LICENSE("GPL");                              
//模块许可声明 

```

`makefile` 文件

```
ifneq ($(KERNELRELEASE),) 
obj-m := mymodules.o       
#obj-m指编译成外部模块 
else 
KERNELDIR := /lib/modules/$(shell uname -r)/build  
#定义一个变量，指向内核目录 
PWD := $(shell pwd)  
modules:         
    $(MAKE) -C $(KERNELDIR) M=$(PWD) modules  
#编译内核模块  
endif 

```

* 执行make检查output

![image-20221128172549751](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221128172549751.png)

<center>
    Fig 3-1: make 执行结果，产出.ko文件
</center>

* 加载内核并查看

![image-20221128172736849](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221128172736849.png)

<center>
    Fig 3-2: 加载内核，系统出现mymodules
</center>

* 查看内核日志信息，执行：\#dmesg –c

![](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221128173003688.png)

<center>
    Fig 3-3: 内核日志信息
</center>

* 卸载动态模块 执行 #rmmod mymodules

![image-20221128173819084](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221128173819084.png)

<center>
    Fig 3-4: 卸载内核，可以看到mymodules已经被移除
</center>


## 预备：篡改系统调用

这里，在实验时出现如下情况：

* 在服务器上执行`insmod xxx.ko` 会崩溃，推测是服务器硬件太差，2核4G的弱配置不支持
* 在高版本Ubuntu 16.04中`insmod xxx.ko` 会一直失败，这是因为syscall table在高版本的虚拟机中读写有额外保护，只读情况比较复杂

因此，以下实验在Ubuntu12.04环境下执行

### 一些尝试

在使用PPT的例子时，先修改系统调用表的地址

![image-20221203114531918](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221203114531918.png)

<center>
    Fig 3-5: 修改系统调用表地址
</center>

这里，系统调用表的地址是只读的

![image-20221203115339670](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221203115339670.png)

<center>
    Fig 3-6: 查询系统调用表地址
</center>

篡改`78 -getoftimeday`时失败，且安装后会显示`killed`并一直被调用。

![image-20221203114031622](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221203114031622.png)

<center>
    Fig 3-7: 78号篡改失败
</center>

编译测试文件

*  gcc -o new modify_new_syscall.c

```
#include<stdio.h>
#include<sys/time.h>
#include<unistd.h>
int main()
{
int ret=syscall(78,10,20); //after modify syscall78
printf("%d\n",ret);
return 0;
}
```

* gcc -o old modify_old_syscall.c

```
#include<stdio.h>
#include<sys/time.h>
#include<unistd.h>
int main()
{
struct timeval tv;
syscall(78,&tv,NULL); //before modify syscall 78 :gettimeofday
printf("tv_sec:%d\n",tv.tv_sec);
printf("tv_usec:%d\n",tv.tv_usec);
return 0;
}
```

得到如下结果

![image-20221203114234068](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221203114234068.png)

<center>
    Fig 3-6: 测试结果，ret返回为-1说明篡改失败
</center>

说明篡改失败，推测同样的只读权限不允许修改。

### 正式实验结果

经查找资料 [<sup>1</sup>](#refer-anchor-1), 尝试在**空余的系统调用号** 中进行篡改操作

Linux操作系统利用系统调用号来标识系统调用，每一个系统调用对应唯一的一个系统调用号。使用`sudo find / -name unistd_32.h`查找预留系统调用号。

![image-20221203120945899](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221203120945899.png)

<center>
    Fig 3-7: 系统调用情况
</center>


其中223号系统调用没有使用，可以篡改此调用号。

### 系统调用表

Linux操作系统利用函数指针数组保存系统调用服务程序的地址，这个数组成为系统调用表（sys_call_table）。使用/proc/kallsyms获取系统调用表的首地址，命令为sudo cat /proc/kallsyms | grep sys_call_table

![image-20221203115339670](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221203115339670.png)

<center>
    Fig 3-8: 系统调用表的首地址
</center>

R代表的是只读属性，所以为了能顺利篡改系统调用的地址，需要提前对控制寄存器CR0 的第16 位WP（写保护位）进行修改，在程序退出时则进行还原。

修改控制寄存器的函数为clear_cr0，代码如下。

```
unsigned int clear_cr0(void)        
{
        unsigned int cr0 = 0;
        unsigned int ret;
        //move the value in reg cr0 to reg rax
        //movl moves a 32-bits operand
        //movq moves a 64-bits operand
        //rax is a 64-bits register
        //an assembly language code
        //asm volatile ("movl %%cr0, %%eax" : "=a"(cr0));//32-bits        
        asm volatile ("movq %%cr0, %%rax" : "=a"(cr0));        //64-bits
        ret = cr0;
        //var cr0 is rax        
        cr0 &= 0xfffeffff; //set 0 to the 17th bit
        //asm volatile ("movl %%eax, %%cr0" :: "a"(cr0));//32-bits
        //note that cr0 above is a variable while cr0 below is a reg.        
        asm volatile ("movq %%rax, %%cr0" :: "a"(cr0));        
        return ret;
}

//recover the value of WP 
void setback_cr0(unsigned int val)
{        
        //asm volatile ("movl %%eax, %%cr0" :: "a"(val));//32-bits
        asm volatile ("movq %%rax, %%cr0" :: "a"(val));//64-bits
}


```

编写自己的系统调用函数my_sys_func，输出我的班级学号，返回输入参数之和。

```
static int my_sys_func(int a,int b,int c)
{
        printk("Change syscall successfully!\nReturn a+b+c\nBy zhp-cs 94-No 2194313178\n");
        return a+b+c;
}

```

完整代码`modify_syscall`：

```
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/unistd.h>
#include <linux/time.h>
#include <asm/uaccess.h>
#include <linux/sched.h>
#include <linux/kallsyms.h>
//using syscall 223
#define __NR_syscall 223        
unsigned long *sys_call_table;
unsigned int clear_cr0(void);
void setback_cr0(unsigned int val);
static int sys_mycall(int a, int b, int c);
//to save the original value of the register cr0
unsigned long orig_cr0;        
unsigned long *sys_call_table = 0;
//to save the original syscall func
unsigned long old_sys_call_func; 
//set 0 to the 17th bit (WP) in reg cr0
unsigned int clear_cr0(void)        
{
        unsigned int cr0 = 0;
        unsigned int ret;
        //move the value in reg cr0 to reg rax
        //movl moves a 32-bits operand
        //movq moves a 64-bits operand
        //rax is a 64-bits register
        //an assembly language code
        //asm volatile ("movl %%cr0, %%eax" : "=a"(cr0));//32-bits        
        asm volatile ("movq %%cr0, %%rax" : "=a"(cr0));        //64-bits
        ret = cr0;
        //var cr0 is rax        
        cr0 &= 0xfffeffff; //set 0 to the 17th bit
        //asm volatile ("movl %%eax, %%cr0" :: "a"(cr0));//32-bits
        //note that cr0 above is a variable while cr0 below is a reg.        
        asm volatile ("movq %%rax, %%cr0" :: "a"(cr0));        
        return ret;
}

//recover the value of WP 
void setback_cr0(unsigned int val)
{        
        //asm volatile ("movl %%eax, %%cr0" :: "a"(val));//32-bits
        asm volatile ("movq %%rax, %%cr0" :: "a"(val));//64-bits
}

//my syscall function
static int sys_mycall(int a,int b,int c)
{
        printk("Change syscall successfully!\nReturn a+b+c\nBy zhp-cs 94-No 2194313178\n");
        return a+b+c;
}

static int __init init_addsyscall(void)
{
        printk("Begin changing syscall...\n");
        //Automatically get sys_call_table address
        sys_call_table = (unsigned long *)kallsyms_lookup_name("sys_call_table");
        //使用了kallsyms_lookup_name("sys_call_table");自动获取地址
        //print sys_call_table address
        printk("sys_call_table: 0x%p\n", sys_call_table);
        //save original syscall func
        old_sys_call_func = (int(*)(void))(sys_call_table[__NR_syscall]);        
        //modify the value of WP in CR0 
        orig_cr0 = clear_cr0();        
        //change the syscall address
        sys_call_table[__NR_syscall] = (unsigned long)&sys_mycall;        
        //setback the value of WP in CR0 
        //to read only
        setback_cr0(orig_cr0);        
        return 0;
}


static void __exit exit_addsyscall(void)
{
        //modify the value of WP in CR0
        orig_cr0 = clear_cr0();        
        //change the syscall address
        sys_call_table[__NR_syscall] =old_sys_call_func;        
        //setback the value of WP in CR0 
        //to read only
        setback_cr0(orig_cr0);
        printk("Recovering syscall...\n");        
}

module_init(init_addsyscall);
module_exit(exit_addsyscall);
MODULE_LICENSE("GPL");

```

得到输出结果 78+10+20=108

![image-20221203131823583](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221203131823583.png)

<center>
    Fig 3-9: 篡改输出结果
</center>

查看日志，输出了我的名字班级学号，说明篡改成功

![image-20221203131948859](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221203131948859.png)

<center>
    Fig 3-10: 篡改日志
</center>

## 字符设备驱动

编写一个简单的字符设备驱动程序，该字符设备并不驱动特定的硬件, 而是用内核空间模拟字符设备，要求该字符设备包括以下几个基本操作，打开、读、写和释放，并编写测试程序用于测试所编写的字符设备驱动程序。在此基础上，编写程序实现对该字符设备的同步操作。

![dev](C:\Users\Administrator\Desktop\os_hw\os_exp\exp3\dev.jpg)

<center>
    Fig 3-11: Overview设备驱动
</center>

在Linux内核中：

* 使用cdev结构体来描述字符设备;

* 通过其成员dev_t来定义设备号（分为主、次设备号）以确定字符设备的唯一性;

* 通过其成员file_operations来定义字符设备驱动提供给VFS的接口函数，如常见的open()、read()、write()等;

`globalval`程序

```
#include<linux/module.h> 
#include<linux/init.h> 
#include<linux/fs.h> 
#include<linux/uaccess.h> 
#include<linux/wait.h> 
#include<linux/semaphore.h> 
#include<linux/sched.h> 
#include<linux/cdev.h> 
#include<linux/types.h> 
#include<linux/kdev_t.h>
#include<linux/device.h> 
#define MAXNUM 100 
#define MAJOR_NUM 456 //主设备号 ，没有被使用
//设备的结构体
struct Scull_Dev{ 
    struct cdev devm; //字符设备 
    struct semaphore sem; //信号量，实现读写时的 PV 操作 
    wait_queue_head_t outq; //等待队列，实现阻塞操作 
    int flag; //阻塞唤醒标志
    char buffer[MAXNUM+1]; //字符缓冲区 
    char *rd,*wr,*end; //读，写，尾指针 
}; 
//虚拟字符设备globalvar
struct Scull_Dev globalvar; 
static struct class *my_class;
//MAJOR_NUM=456,主设备号
int major=MAJOR_NUM; 
//函数声明：读、写、打开、释放
static ssize_t globalvar_read(struct file *,char *,size_t ,loff_t *); 
static ssize_t globalvar_write(struct file *,const char *,size_t ,loff_t *); 
static int globalvar_open(struct inode *inode,struct file *filp); 
static int globalvar_release(struct inode *inode,struct file *filp); 
//字符设备的数据接口，将文件的读、写、打开、释放等操作映射为相应的函数。 
struct file_operations globalvar_fops = 
{ 
    .read=globalvar_read, 
    .write=globalvar_write, 
    .open=globalvar_open, 
    .release=globalvar_release, 
}; 
/*
设备初始化：
调用内核函数register_chrdev把驱动程序的基本入口点
指针存放在内核的字符设备地址表中，在用户进程对该设备
执行系统调用时提供入口地址
*/
static int globalvar_init(void) 
{ 
    int result = 0; 
    int err = 0; 
    //MKDEV获取设备在设备表中的位置
    //major是主设备号，minor是次设备号
    //这里是新定义一个设备号（）
    dev_t dev = MKDEV(major, 0); 
    if(major) 
    { 
        //静态申请设备编号
        //第一个参数表示设备号，第二个参数表示注册的此设备数目，
        //第三个表示设备名称。
        result = register_chrdev_region(dev, 1, "charmem"); 
    } 
    else 
    { 
        //动态分配设备号
        //第一个参数保存生成的设备号，第二个参数表示次设备号的基准，
        //即从哪个次设备号开始分配，第三个表示注册的此设备数目，
        //第四个表示设备名称。
        result = alloc_chrdev_region(&dev, 0, 1, "charmem"); 
        major = MAJOR(dev);
    } 
    //返回值：小于0，则自动分配设备号错误。否则分配得到的设备号就被&dev带出来。
    if(result < 0) 
        return result; 
    //将struct cdev类型的结构体变量和file_operations结构体进行绑定。
    cdev_init(&globalvar.devm, &globalvar_fops); 
    //cdev中的struct module *owner;填充时，值要为 THIS_MODULE，表示模块
    globalvar.devm.owner = THIS_MODULE; 
    //向内核里面添加一个驱动，注册驱动
    //第一个输入参数代表即将被添加入Linux内核系统的字符设备
    //第二个输入参数是dev_t类型的变量，此变量代表设备的设备号
    //第三个输入参数是无符号的整型变量，代表想注册设备的设备号的范围
    //如果成功，则返回0，如果失败，则返回ENOMEM, ENOMEM的被定义为12。
    -----------------------------------
    err = cdev_add(&globalvar.devm, dev, 1); 
    if(err) 
        printk(KERN_INFO "Error %d adding char_mem device", err); 
    else
    { 
        //设备注册成功
        printk("globalvar register success\n"); 
        sema_init(&globalvar.sem,1); //初始化信号量
        init_waitqueue_head(&globalvar.outq); //初始化等待队列
        globalvar.rd = globalvar.buffer; //读指针 
        globalvar.wr = globalvar.buffer; //写指针 
        globalvar.end = globalvar.buffer + MAXNUM;//缓冲区尾指针 
        globalvar.flag = 0; // 阻塞唤醒标志置 0 
    } 
    ---------------------------------------
    //创建设备文件
    my_class = class_create(THIS_MODULE, "chardev0"); 
    device_create(my_class, NULL, dev, NULL, "chardev0");
    return 0; 
} 
//.open=globalvar_open, 函数映射
static int globalvar_open(struct inode *inode,struct file *filp) 
{ 
    //如果该模块处于活动状态且对它引用计数加1操作正确则返回1，否则返回0.
    try_module_get(THIS_MODULE);//模块计数加一 
    printk("This chrdev is in open\n"); 
    return(0); 
} 
 
static int globalvar_release(struct inode *inode,struct file *filp) 
{ 
    module_put(THIS_MODULE); //模块计数减一 
    printk("This chrdev is in release\n"); 
    return(0); 
}
static void globalvar_exit(void) 
{ 
    //注销设备
    device_destroy(my_class, MKDEV(major, 0)); 
    class_destroy(my_class);
    cdev_del(&globalvar.devm); 
    unregister_chrdev_region(MKDEV(major, 0), 1); //注销设备 
} 
 
static ssize_t globalvar_read(struct file *filp,char *buf,size_t len,loff_t *off) 
{ 
    //globalvar.flag是阻塞唤醒标志，为0可读
    //条件condition为真时调用这个函数将直接返回0    
    if(wait_event_interruptible(globalvar.outq,globalvar.flag!=0)) //不可读时 阻塞读进程 
    { 
        return -ERESTARTSYS; 
    } 
    if(down_interruptible(&globalvar.sem)) //P 操作 
    { 
        return -ERESTARTSYS; 
    } 
    globalvar.flag = 0; 
    printk("into the read function\n"); 
    printk("the rd is %c\n",*globalvar.rd); //读指针 
    //读指针小于写指针
    if(globalvar.rd < globalvar.wr) 
        len = min(len,(size_t)(globalvar.wr - globalvar.rd)); //更新读写长度 
    else 
        len = min(len,(size_t)(globalvar.end - globalvar.rd)); 
    printk("the len is %d\n",len); 
    //copy_to_user()完成用户空间到内核空间的复制，函数copy_from_user()完成内核空间到
    //用户空间的复制。如果数据拷贝成功，则返回零；否则，返回没有拷贝成功的数据字节数。
    if(copy_to_user(buf,globalvar.rd,len)) 
    { 
        printk(KERN_ALERT"copy failed\n"); 
        up(&globalvar.sem); //V操作
        return -EFAULT; 
    } 
    printk("the read buffer is %s\n",globalvar.buffer); 
    globalvar.rd = globalvar.rd + len;
    if(globalvar.rd == globalvar.end) 
        globalvar.rd = globalvar.buffer; //字符缓冲区循环 
    up(&globalvar.sem); //V 操作 
    return len; 
} 
//写入
static ssize_t globalvar_write(struct file *filp,const char *buf,size_t len,loff_t *off) 
{ 
    if(down_interruptible(&globalvar.sem)) //P 操作 
    { 
        return -ERESTARTSYS; 
    } 
    if(globalvar.rd <= globalvar.wr) 
        len = min(len,(size_t)(globalvar.end - globalvar.wr)); 
    else 
        len = min(len,(size_t)(globalvar.rd-globalvar.wr-1)); 
    printk("the write len is %d\n",len); 
    //从用户空间写入内核空间
    //该字符设备并不驱动特定的硬件, 而是用内核空间模拟字符设备
    if(copy_from_user(globalvar.wr,buf,len)) 
    { 
        up(&globalvar.sem); //V 操作 
        return -EFAULT; 
    } 
    printk("the write buffer is %s\n",globalvar.buffer); 
    printk("the len of buffer is %d\n",strlen(globalvar.buffer)); 
    globalvar.wr = globalvar.wr + len; 
    if(globalvar.wr == globalvar.end) 
        globalvar.wr = globalvar.buffer; //循环 
    up(&globalvar.sem); //V 操作 
    globalvar.flag=1; //条件成立，可以唤醒读进程 
    wake_up_interruptible(&globalvar.outq); //唤醒读进程 
    return len; 
} 
 
module_init(globalvar_init); 
module_exit(globalvar_exit);
MODULE_LICENSE("GPL");

```

### 程序详解：

* `struct Scull_Dev globalvar;` 定义了一个虚拟字符设备驱动globalvar。

*  声明虚拟字符设备的读、写、打开和释放操作函数

```
static ssize_t globalvar_read(struct file *,char *,size_t ,loff_t *); 
static ssize_t globalvar_write(struct file *,const char *,size_t ,loff_t *); 
static int globalvar_open(struct inode *inode,struct file *filp); 
static int globalvar_release(struct inode *inode,struct file *filp); 
```

* 将文件的读、写、打开、释放等操作映射为虚拟字符设备的函数

```
struct file_operations globalvar_fops = 
{ 
    .read=globalvar_read, 
    .write=globalvar_write, 
    .open=globalvar_open, 
    .release=globalvar_release, 
}; 

```

*  对设备进行初始化

1. 函数`static int globalvar_init(void)`对设备进行初始化。一是获取设备号，使用`dev_t dev = MKDEV(major, 0);`定义一个设备号，申请或分配设备号.

```
//MKDEV获取设备在设备表中的位置
    //major是主设备号，minor是次设备号
    //这里是新定义一个设备号（）
    dev_t dev = MKDEV(major, 0); 
if(major) 
    { 
        //静态申请设备编号
        //第一个参数表示设备号，第二个参数表示注册的此设备数目，
        //第三个表示设备名称。
        result = register_chrdev_region(dev, 1, "charmem"); 
    } 
    else 
    { 
        //动态分配设备号
        //第一个参数保存生成的设备号，第二个参数表示次设备号的基准，
        //即从哪个次设备号开始分配，第三个表示注册的此设备数目，
        //第四个表示设备名称。
        result = alloc_chrdev_region(&dev, 0, 1, "charmem"); 
        major = MAJOR(dev);
    } 
    //返回值：小于0，则自动分配设备号错误。否则分配得到的设备号就被&dev带出来。
    if(result < 0) 
    return result;
//用class_create和device_create创建设备文件。
my_class = class_create(THIS_MODULE, "chardev0"); 
device_create(my_class, NULL, dev, NULL, "chardev0");
```

2. 向内核里面注册驱动

​	第一个输入参数代表即将被添加入Linux内核系统的字符设备，第二个输入参数是dev_t类型的变量，此变量代表设备的设备号，第三个输入参数是无符号的整型变量，代表想注册设备的设备号的范围。如果成功，则返回	0，如果失败，则返回ENOMEM, ENOMEM的被定义为12。

```
err = cdev_add(&globalvar.devm, dev, 1); 
if(err) 
    printk(KERN_INFO "Error %d adding char_mem device", err); 
else
{ 
    //设备注册成功
    printk("globalvar register success\n"); 
    sema_init(&globalvar.sem,1); //初始化信号量
    init_waitqueue_head(&globalvar.outq); //初始化等待队列
    globalvar.rd = globalvar.buffer; //读指针 
    globalvar.wr = globalvar.buffer; //写指针 
    globalvar.end = globalvar.buffer + MAXNUM;//缓冲区尾指针 
    globalvar.flag = 0; // 阻塞唤醒标志置 0 
} 

```

* 打开和释放

打开时模块计数加一

```
static int globalvar_open(struct inode *inode,struct file *filp) 
{ 
    //如果该模块处于活动状态且对它引用计数加1操作正确则返回1，否则返回0.
    try_module_get(THIS_MODULE);//模块计数加一 
    printk("This chrdev is in open\n"); 
    return(0); 
} 
```

释放时减一

```
static int globalvar_release(struct inode *inode,struct file *filp) 
{ 
    module_put(THIS_MODULE); //模块计数减一 
    printk("This chrdev is in release\n"); 
    return(0); 
}
```

* 注销设备操作

具体为用device_destroy注销创建的设备，用class_destroy注销设备类，用cdev_del释放cdev结构体空间，用unregister_chrdev_region来注销设备号。

```
static void globalvar_exit(void) 
{ 
    //注销设备
    device_destroy(my_class, MKDEV(major, 0)); 
    class_destroy(my_class);
    cdev_del(&globalvar.devm); 
    unregister_chrdev_region(MKDEV(major, 0), 1); //注销设备
} 
```

* 读操作

`globalvar.flag`标志当前是否可读，若不可读，则用`wait_event_interruptible`将其挂起到等待队列`globalvar.outq`。如果可以读，则使用`down_interruptible(&globalvar.sem)`进行P操作。接下来更新读指针，len是读的字节数，如果读指针小于写指针，表明新写入了内容，应该读取从写指针到读指针的内容，即`len = min(len,(size_t)(globalvar.wr - globalvar.rd))`。如果如指针大于等于写指针，表明在循环缓冲区中写指针已经过了一次循环，应令`len = min(len,(size_t)(globalvar.end - globalvar.rd))`将当前读指针到结尾的内容读完，下一次再读到写指针。用`copy_to_user(buf,globalvar.rd,len)`函数将内核空间的数据读取出来，并更新读指针位置，如果读指针在缓冲区末尾则将其循环地置为缓冲区首部。最后进行V操作退出临界区。

```
static ssize_t globalvar_read(struct file *filp,char *buf,size_t len,loff_t *off) 
{ 
    //globalvar.flag是阻塞唤醒标志，为0可读
    //条件condition为真时调用这个函数将直接返回0    
    if(wait_event_interruptible(globalvar.outq,globalvar.flag!=0)) //不可读时 阻塞读进程 
    { 
        return -ERESTARTSYS; 
    } 
    if(down_interruptible(&globalvar.sem)) //P 操作 
    { 
        return -ERESTARTSYS; 
    } 
    globalvar.flag = 0; 
    printk("into the read function\n"); 
    printk("the rd is %c\n",*globalvar.rd); //读指针 
    //读指针小于写指针
    if(globalvar.rd < globalvar.wr) 
        len = min(len,(size_t)(globalvar.wr - globalvar.rd)); //更新读写长度 
    else 
        len = min(len,(size_t)(globalvar.end - globalvar.rd)); 
    printk("the len is %d\n",len); 
    //copy_to_user()完成用户空间到内核空间的复制，函数copy_from_user()完成内核空间到
    //用户空间的复制。如果数据拷贝成功，则返回零；否则，返回没有拷贝成功的数据字节数。
    if(copy_to_user(buf,globalvar.rd,len)) 
    { 
        printk(KERN_ALERT"copy failed\n"); 
        up(&globalvar.sem); //V操作
        return -EFAULT; 
    } 
    printk("the read buffer is %s\n",globalvar.buffer); 
    globalvar.rd = globalvar.rd + len;
    if(globalvar.rd == globalvar.end) 
        globalvar.rd = globalvar.buffer; //字符缓冲区循环 
    up(&globalvar.sem); //V 操作 
    return len; 
} 
```

* 写操作

写操作和读操作大致相同，只不过读写方向相反。首先用P操作进入临界区，计算写入长度len。如果读指针小于写指针，则可以令`len = min(len,(size_t)(globalvar.end - globalvar.wr))`; 表示从当前写指针到缓冲区末尾都可以写入。如果读指针大于写指针，则只能写入从写指针到读指针之前的位置，即`len = min(len,(size_t)(globalvar.rd-globalvar.wr-1));`，否则会破坏还未读取的内容，读取时也不能读取写指针后面、本应读取到的内容。最后更新写指针，进行V操作退出临界区，通过 `wake_up_interruptible(&globalvar.outq)`唤醒读进程。采用这种方式实际上写入的优先级更高。

```
static ssize_t globalvar_write(struct file *filp,const char *buf,size_t len,loff_t *off) 
{ 
    if(down_interruptible(&globalvar.sem)) //P 操作 
    { 
        return -ERESTARTSYS; 
    } 
    if(globalvar.rd <= globalvar.wr) 
        len = min(len,(size_t)(globalvar.end - globalvar.wr)); 
    else 
        len = min(len,(size_t)(globalvar.rd-globalvar.wr-1)); 
    printk("the write len is %d\n",len); 
    //从用户空间写入内核空间
    //该字符设备并不驱动特定的硬件, 而是用内核空间模拟字符设备
    if(copy_from_user(globalvar.wr,buf,len)) 
    { 
        up(&globalvar.sem); //V 操作 当进程希望释放内核信号量锁时，就调用up()函数。
        return -EFAULT; 
    } 
    printk("the write buffer is %s\n",globalvar.buffer); 
    printk("the len of buffer is %d\n",strlen(globalvar.buffer)); 
    globalvar.wr = globalvar.wr + len; 
    if(globalvar.wr == globalvar.end) 
        globalvar.wr = globalvar.buffer; //循环 
    up(&globalvar.sem); //V 操作 
    globalvar.flag=1; //条件成立，可以唤醒读进程 
    wake_up_interruptible(&globalvar.outq); //唤醒读进程 
    return len; 
} 
```

### 测试demo

实现两个窗口的通信，编写既能读又能写的程序

```
#include<sys/types.h>
#include<unistd.h>
#include<sys/stat.h>
#include<stdio.h>
#include<fcntl.h>
#include<string.h>
#include<stdlib.h>
int fd,i;
char msg[101];
void printbar()
{
    printf("---------------------------------------------\n");
}
int main()
{
    printbar();
    printf("本程序可以对设备进行读写，");
    fd = open("/dev/chardev0",O_RDWR,S_IRUSR|S_IWUSR);
    int operation;
    while(1)
    {
        printf("请选择您的操作：按1读取信息，按2写入信息，按3退出。\n");
        printbar();
        scanf("%d",&operation);
        getchar();
        switch (operation)
        {
            case 1:
            {
                printf("您选择了：读取信息。\n");
                if(fd!=-1)
                {
                    {
                        for(i=0;i<101;i++)  //初始化
                            msg[i]='\0';
                        printf("读取的信息是：");
                        read(fd,msg,100);
                        printf("%s\n",msg);
                        printbar();
                    }
                }
                else
                {
                    printf("设备打开失败！%d\n",fd);
                    printbar();
                    exit(1);
                }
                continue;
            }
            case 2:
            {
                printf("您选择了：写入信息。\n");
                if(fd!=-1)
                {
                        printf("请输入要写入的信息:\n");
                        scanf("%s",msg);
                        write(fd,msg,strlen(msg));
                        printf("您写入了%s。\n",msg);
                        printbar();
                }
                else
                {
                    printf("设备打开失败！\n");
                    printbar();
                    exit(1);
                }
                continue;
            }
            case 3:
            {
                printf("再见！\n");
                printbar();
                return 0;
            }
            default:
            {
                printf("不支持该操作！\n");
                printbar();
                continue;
            }
        }
    }
}
```

启用sudo权限，结果如下：

左边读取，右边写入。右边还未输入完成，左边在等待状态。

![image-20221203150502353](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221203150502353.png)

<center>
    Fig 3-12: 实验结果1-左边聊天等待右边输入
</center>

写入后，左边能够读取信息`12345`

![image-20221203150601586](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221203150601586.png)

<center>
    Fig 3-13: 实验结果2-左边聊天接收右边输入
</center>

查看日志

![image-20221203150727351](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221203150727351.png)

<center>
    Fig 3-14: 日志结果
</center>

## 遇到的问题以及解决

* makefile 执行时，一定要检查命令行是否为TAB而不是空格，否则出现 `Makefile:7: missing separator. Stop` 报错

* 篡改系统调用由于Ubuntu更新缘故，高版本内核的syscall table是只读的，因此高版本的无法`insmod modify_syscall.ko` 。本实验采用12.04版本的Ubuntu



## 引用文献

<div id="refer-anchor-1"></div>
- [1] [CSDN]    (https://blog.csdn.net/qq_34258344/article/details/103228607)

`	`
