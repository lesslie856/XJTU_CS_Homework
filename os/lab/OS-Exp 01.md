# OS-Exp 01: 华为云上 OpenEuler 部署

## 1. 实验要求

### 进程相关编程实验

**a)** **观察进程调度，了解进程调度的过程，了解孤儿进程和僵尸进程的区别是什么**

**b)** **观察进程调度中的全局变量改变，输出父子进程共享变量地址了解物理地址与虚地址概念**

**c)** **在子进程中调用system函数**

**d)** **在子进程中调用exec族函数**

* 熟悉操作命令、编辑、编译、运行程序。完成操作系统原理课程教材P103作业 3.7 （采用图3-32所示的程序）的运行验证，多运行程序几次观察结果；去除wait后再观察结果并进行理论分析。
* 扩展图3-32的程序：
  1. 添加一个全局变量并在父进程和子进程中对这个变量做不同操作，输出操作结果并解释，同时输出两种变量的地址观察并分析；
  2. 在return前增加对全局变量的操作并输出结果，观察并解释；
  3. 修改程序体会在子进程中调用system函数和在子进程中调用exec族函数执行自己写的一段程序，在此程序中输出进程PID进行比较并说明原因



### 线程相关编程实验

1. 在进程中给一变量赋初值并创建两个线程；
2. 在两个线程中分别对此变量循环五千次以上做不同的操作并输出结果 ；
3. 多运行几遍程序观察运行结果，如果发现每次运行结果不同，请解释原因并修改程序解决，考虑如何控制互斥和同步；
4. 将任务一中第一个实验调用system函数和调用exec族函数改成在线程中实现，观察运行结果输出进程PID与线程TID进行比较并说明原因。

## 实验

### origin code test

原有程序--origin.c（图3-32）：

```
# include <sys/types.h>
# include <stdio.h>
# include <unistd.h>

int main()
{
    pid_t pid,pid1;
    //fork a child process
    printf("\n");
    pid = fork();

    if(pid < 0)
    {
        //error occured
        fprintf(stderr, "Fork Failed");
        return 1;
    }
    else if (pid == 0)
    {
        //child process
        pid1 = getpid();
        printf("A child: pid = %d\n",pid);//A
        printf("B child: pid1 = %d\n",pid1);//B
    }
    else 
    {
        //parent process
        pid1 = getpid();
        printf("C parent: pid = %d\n",pid);//C
        printf("D parent: pid1 = %d\n",pid1);//D
        wait(NULL);           
    }
    printf("------------\n");
    return 0;
}

```

得到输出结果

![image-20221024163844896](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221024163844896.png)

<center>
    Fig1. Origin结果
</center>



除去`wait(NULL)`执行，得到如下结果

![image-20221024164033288](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221024164033288.png)

<center>
    Fig2. nowait结果
</center>



#### 分析结果

1. 执行顺序会有所变化，可以是`CADB` 也可以是`CDAB`
2. `fork` 执行后，系统会同时执行父进程和子进程。在**父进程**中，`fork`返回子进程id，即父进程中的`pid=子进程id`；而在子进程中`fork`返回**0**来声明是子进程；而`pid1`直接反应当前进程id。 综上所述，A处恒为0，BC恒为子进程id，D为父进程id（差异可由id大小反应，子进程比父进程大1）。
3. 从结果来看，`wait()`有无都没改变实验结果。`wait()`的作用是让父进程挂起等待子进程结束。所以只有将`wait()`放在父进程打印语句之前才能确保子进程先输出。



![image-20221024165136084](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221024165136084.png)

<center>
    Fig3. fork函数示意
</center>



![image-20221024165445809](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221024165445809.png)

<center>
    Fig4. vim编辑将wait()添加到打印之前，保证了子进程先输出
</center>
### 全局变量test

定义`global`全局变量，子进程+1，父进程+2，`global.c`如下：

```
# include <sys/types.h>
# include <stdio.h>
# include <unistd.h>

int global = 0;

int main()
{
    pid_t pid,pid1;
    //fork a child process
    printf("\n");
    pid = fork();

    if(pid < 0)
    {
        //error occured
        fprintf(stderr, "Fork Failed");
        return 1;
    }
    else if (pid == 0)
    {
        //child process
        pid1 = getpid();
        global+=1;
        printf("A child: pid = %d\n",pid);//A
        printf("B child: pid1 = %d\n",pid1);//B
        printf("Global: %d\n",global);
        printf("address_child:%p\n",&global);
    }
    else 
    {
        //parent process
        pid1 = getpid();
        global+=2;
        printf("C parent: pid = %d\n",pid);//C
        printf("D parent: pid1 = %d\n",pid1);//D
        printf("Global: %d\n",global);
        printf("address_parent:%p\n",&global);
        wait(NULL);           
    }
    printf("%p\n",&global);
    return 0;
}
```

编译得到结果：

![image-20221101152113492](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221101152113492.png)

<center>
    Fig.5 全局变量结果
</center>
**可以看到在子进程和父进程中，`global`的自增是两个独立的过程，即自增执行的物理空间不一样，导致打印的值也不一样, 而打印出的地址一样，说明子进程继承的是父进程刚开始全局变量的值**



继续在`return`前加上 `global+=3`,并打印

```
# include <sys/types.h>
# include <stdio.h>
# include <unistd.h>

int global = 0;

int main()
{
    pid_t pid,pid1;
    //fork a child process
    printf("\n");
    pid = fork();

    if(pid < 0)
    {
        //error occured
        fprintf(stderr, "Fork Failed");
        return 1;
    }
    else if (pid == 0)
    {
        //child process
        pid1 = getpid();
        global+=1;
        printf("A child: pid = %d\n",pid);//A
        printf("B child: pid1 = %d\n",pid1);//B
        printf("Global: %d\n",global);
    }
    else 
    {
        //parent process
        pid1 = getpid();
        global+=2;
        printf("C parent: pid = %d\n",pid);//C
        printf("D parent: pid1 = %d\n",pid1);//D
        printf("Global: %d\n",global);
        
        wait(NULL);           
    }
    global += 3;
    printf("Global: %d\n",global);
    return 0;
}

```

![image-20221024173754125](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221024173754125.png)

<center>
    Fig.5 全局变量结果，return前操作
</center>
**可以验证前面所说的，两个进程发生在不同的空间，而且由于带有`wait()`函数，父进程总会等待子进程结束才继续执行，即`Global:5`总会出现在最后**



### System和exec函数

设计`my_code.c`输出

```
# include<stdio.h>
# include <sys/types.h>
int main()
{
    pid_t pid = getpid();
    printf("\n在子进程中调用system函数，pid为%d\n",pid);
}

```

system:

```
# include <sys/types.h>
# include <stdio.h>
# include <unistd.h>

int main()
{
    pid_t pid,pid1;
    //fork a child process
    pid = fork();

    if(pid < 0)
    {
        //error occured
        fprintf(stderr, "Fork Failed");
        return 1;
    }
    else if (pid == 0)
    {
        //child process
        // printf("A child: pid = %d\n",pid);//A
        pid1 = getpid();
        printf("B child: pid1 = %d\n",pid1);//B
        system("./my_code");
        printf("SIGN OF CHILD PROCESS\n");
        
    }
    else 
    {
        //parent process
        pid1 = getpid();
        // printf("C parent: pid = %d\n",pid);//C
        printf("D parent: pid1 = %d\n",pid1);//D
        wait(NULL);           
    }

    return 0;
}

```

得到结果

![image-20221102180207806](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221102180207806.png)

<center>
    Fig.6 system调用结果
</center>

**分析：** `system`输出的子进程的pid-7222与原本的pid1-7221**不一样**，因为`system()`函数先执行了`fork()`函数，然后新产生的子进程立刻执行了`exec()`函数，产生了一个新进程，所以新进程的pid等与原进程不同。



exec:

```
# include <sys/types.h>
# include <stdio.h>
# include <unistd.h>

int main()
{
    pid_t pid,pid1;
    //fork a child process
    pid = fork();

    if(pid < 0)
    {
        //error occured
        fprintf(stderr, "Fork Failed");
        return 1;
    }
    else if (pid == 0)
    {
        //child process
        // printf("A child: pid = %d\n",pid);//A
        pid1 = getpid();
        printf("B child: pid1 = %d\n",pid1);//B
        execl("./my_code",NULL);
        printf("SIGN OF CHILD PROCESS\n");
        
    }
    else 
    {
        //parent process
        pid1 = getpid();
        // printf("C parent: pid = %d\n",pid);//C
        printf("D parent: pid1 = %d\n",pid1);//D
        wait(NULL);           
    }

    return 0;
}

```

仍执行相同的`my_code`，得到结果

![image-20221102180042912](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221102180042912.png)

<center>
    Fig.7 exec调用结果
</center>



可以看到`my_code`仍然执行，打印出了与子进程一样的PID，但后面的 `SIGN OF CHILD PROCESS`没有打印出来，这是因为并没有产生新的进程，只是用调用的内容替换掉了原来的内容。



> Reference
>
> *system*是用*shell*来调用程序*=fork+exec+waitpid*，而*exec*是直接让你的程序代替用来的程序运行。
>
> https://blog.csdn.net/goodlixueyong/article/details/6315596



### 多线程实验

为便于观察，5k次循环改成10次

代码如下：

```
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
int a = 5000; // 全局变量
void run() {  int i=0; while(i<10){ a=a+10;i++;printf("%d\n",a);}    }
void run2() {  int i=0;  while(i<10){ a=a+5;i++;printf("%d\n",a);}   }
int main() {    
 	int tmp1, tmp2;
	pthread_t thread1, thread2;	
	int ret_thrd1, ret_thrd2;
//线程1--------------------------
	ret_thrd1 = pthread_create(&thread1, NULL, (void *)&run,  NULL);
	if (ret_thrd1 != 0) {
		printf("thread1 create error\n");
	 } 
	else {
		printf("thread1 create success\n");
	}
	pthread_join(thread1,NULL);
//线程2--------------------------
	ret_thrd2 = pthread_create(&thread2, NULL, (void *)&run2, NULL);
	if (ret_thrd2 != 0) {
	 	printf("thread2 create error\n");
	 } else {
		printf("thread2 create success\n");
	}
	pthread_join(thread2,NULL);
}



```



![image-20221102195739983](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221102195739983.png)

<center>
    Fig.8 pthread结果
</center>

经过多次实验发现实验结果相同，程序始终按照顺序增加：thrd1`+10` thrd2`+20`.两个线程之间始终保持同步。

而如果取消同步操作，即去除`pthread_join(thread1,NULL)`后，同步关系被打破，thrd2会开始抢占，操作系统内部实现了线程之间的互斥操作，thread2和thread1同一时间只有一个可以对临界区（critical section）进行操作。

![image-20221102200154812](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221102200154812.png)

<center>
    Fig.9 pthread互斥结果
</center>

pthread_join()函数的作用：以阻塞的方式等待thread指定的线程结束。当函数返回时，被等待线程的资源被收回。如果线程已经结束，那么该函数会立即返回。

在代码中，当thread1运行结束之后对资源进行回收，在这之前不允许thread2对全局变量进行操作，这实现了进程之间的互斥。



**多线程的system和exec**

```
#include <stdio.h>
#include <pthread.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/syscall.h>

void *thread_one()
{
    printf("thread_one:int %d main process, the tid=%lu,pid=%ld\n",getpid(),pthread_self(),syscall(SYS_gettid));
}

void *thread_two()
{
    printf("thread two:int %d main process, the tid=%lu,pid=%ld\n",getpid(),pthread_self(),syscall(SYS_gettid));
}

int main(int argc, char *argv[])
{
    pid_t pid;
    pthread_t tid_one,tid_two;
    if((pid=fork())==-1)
    {
        perror("fork");
        exit(EXIT_FAILURE);
    }
    else if(pid==0)
    {
        pthread_create(&tid_one,NULL,(void *)thread_one,NULL);
        pthread_join(tid_one,NULL);
        system("./my_code");
        printf("SIGN OF CHILD PROCESS\n");
    }
    else
    {
        pthread_create(&tid_two,NULL,(void *)thread_two,NULL);
        pthread_join(tid_two,NULL);
    }
    
    wait(NULL);
    return 0;
}
```

![image-20221102205851198](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221102205851198.png)

system函数还是和之前一样，会产生新进程，进程的PID都不一样



而在exec中

```
#include <stdio.h>
#include <pthread.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/syscall.h>

void *thread_one()
{
    printf("thread_one:int %d main process, the tid=%lu,pid=%ld\n",getpid(),pthread_self(),syscall(SYS_gettid));
}

void *thread_two()
{
    printf("thread two:int %d main process, the tid=%lu,pid=%ld\n",getpid(),pthread_self(),syscall(SYS_gettid));
}

int main(int argc, char *argv[])
{
    pid_t pid;
    pthread_t tid_one,tid_two;
    if((pid=fork())==-1)
    {
        perror("fork");
        exit(EXIT_FAILURE);
    }
    else if(pid==0)
    {
        pthread_create(&tid_one,NULL,(void *)thread_one,NULL);
        pthread_join(tid_one,NULL);
        execl("./my_code",NULL);
        printf("SIGN OF CHILD PROCESS\n");
    }
    else
    {
        pthread_create(&tid_two,NULL,(void *)thread_two,NULL);
        pthread_join(tid_two,NULL);
    }
    
    wait(NULL);
    return 0;
}
```

![image-20221102210435773](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221102210435773.png)

因为没有产生新的进程，只是用调用的内容替换掉了原来的内容，一样104843一样。



**分析**：可以看到多线程的tid都一样，而pid不一样。getpid()得到的是进程的pid，在内核中，每个线程都有自己的PID，要得到线程的PID,必须用syscall(SYS_gettid); pthread_self函数获取的是线程ID，线程ID在某进程中是唯一的，在不同的进程中创建的线程可能出现ID值相同的情况。

![img](https://img-blog.csdnimg.cn/20210125181754115.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2Nob3VtaW4=,size_16,color_FFFFFF,t_70)

> Reference
>
> https://www.cnblogs.com/lakeone/p/3789117.html
