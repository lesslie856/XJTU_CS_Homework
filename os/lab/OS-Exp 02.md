# Exp2: Linux 进程通信与内存管理

## 实验要求：

* 本实验在用户态下，根据教材所学习的操作系统原理，完成Linux下进程通信与内存管理算法的实现，通过实验，进一步理解所学理论知识。

* 进程软中断与管道通信（必做）
* 内存分配与回收、页面置换（二者选一）

## 2.1 进程的软中断通信

### 目的
编程实现进程的创建和软中断通信，通过观察、分析实验现象，深入理解进程及进程在调度执行和内存空间等方面的特点，掌握在POSIX 规范中系统调用的功能和使用

### 实验内容

编制实现软中断通信的程序
使用系统调用fork()创建两个子进程，再用系统调用signal()让父进程捕捉键盘上发出的中断信号（即按delete键），当父进程接收到这两个软中断的某一个后，父进程用系统调用kill()向两个子进程分别发出整数值为16和17软中断信号，子进程获得对应软中断信号，然后分别输出下列信息后终止：
Child process 1 is killed by parent !!
Child process 2 is killed by parent !!
父进程调用wait()函数等待两个子进程终止后，输入以下信息，结束进程执行：
Parent process is killed!!
多运行几次编写的程序，简略分析出现不同结果的原因。

### 实验结果

![image-20221112153209886](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221112153209886.png)

代码如下：

```
#include <stdio.h> 
#include <signal.h>     
#include <unistd.h>  
#include <sys/types.h>

int wait_flag;
void stop(int signum);
int main(){ 
    int pid1, pid2;    
    //alarm(1);                                          
    //signal(3,stop); // ctrl '\'
    signal(2,stop);//  ctrl C
    //signal(14,stop);//alarm
    while((pid1 = fork( )) == -1);                 
    //在父进程里
    if(pid1 > 0) {                               
        //创建子进程2 (与子进程1并列)
        while((pid2 = fork( )) == -1);             
        //在父进程里
        if(pid2 > 0) { 
            wait_flag = 1; 
            //signal(SIGALRM,stop);
            //sleep(5); 
            //while(wait_flag == 1);
            sleep(2);
            // 杀死进程1发中断号16
            //kill的作用不是使进程结束，而是发送信号让子进程自己结束。
            kill(pid1,16);                             
            //--------------------------
            // 杀死进程2发中断号17
            kill(pid2,17); 
                                  
            // 等待第1个子进程1结束的信号            
            wait(0);                                   
            // 等待第2个子进程2结束的信号 
            wait(0);                                    
            printf("\n Parent process is killed !!\n");
            printf("-----------------------------------------\n");
            // 父进程结束  
            
            exit(0);                                                      
            
        } 
        //在pid2里
        else { 
            wait_flag = 1; 
            //补充-----------------------
            // 等待进程2被杀死的中断号17 
            signal(17,stop);
            //--------------------------                                                                                 
            while(wait_flag==1);
            printf("\n Child process 2 is killed by parent !!\n"); 
            
            exit(0);                                                                                                    
        }
    }     
    //在pid1里                                                              
    else {                                                                                                               
        wait_flag = 1;                  
        // 等待进程1被杀死的中断号16                                                                                 
        signal(16,stop);                                                                                                                         
        while(wait_flag==1);
        printf("\n Child process 1 is killed by parent !!\n");                                                        
        exit(0); 
    }                                                                                                                    
}   
void stop(int signum) { 
    wait_flag = 0;
    
    printf("\n %d stop test \n",signum);        
    //--------------------------                                                                    
}  

```

* 先猜想一下这个程序的运行结果。然后按照注释里的要求把代码补充完整，运行程序。或者多次运行，并且Delete/quit,键后，会出现什么结果？分析原因

```
猜想：键盘按下Delete/Quit键后显示“Child process 1 is killed by parent !! Child process 2 is killed by parent !!”，五秒之后显示“Parent process is killed !!”。
实际结果：同猜想一样，在del/quit之前键入其他任何输入都不起作用，因为signal函数并没有对输入的中断进行判断。而且子进程被杀死的顺序不总是一样的。这是因为两个子进程同时运行，并没有先后的区别。
```

![image-20221112153801350](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221112153801350.png)

<center>
    Fig2-1. 软终端softirqs运行结果
</center>

* 请将5秒内中断和5秒后中断的运行结果截图，试对产生该现象的原因进行分析。

在stop（）中添加标志

```
void stop(int signum) { 
    wait_flag = 0;
    printf("\n %d stop test \n",signum); }  
```

分别得到运行结果：

![image-20221113000624462](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221113000624462.png)

<center>
    Fig2-2. 5s后自动运行结果
</center>

![image-20221113000736991](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221113000736991.png)

<center>
    Fig2-3. 5s内del运行结果
</center>

```
分析：sleep 5s后程序正常按顺序执行，因此只检测到自定义信号16和17，且为自动跳转；
而在5s内中断，直接介入子进程，故此时会有信号2
```

* 如果程序运行，界面上显示“Child process 1 is killed by parent !! Child process 2 is killed by parent !!”，五秒之后显示“Parent process is killed !!”，怎样修改程序使得只有接收到相应的中断信号后再发生跳转，执行输出？针对实验过程2，怎样修改的程序？修改前后程序的运行结果是什么？请截图说明。

```
在stop函数中将wait_flag置为0，在kill函数前面加上while(wait_flag==1)，这样只有在接收到中断信号后才会跳转。此时，无论何时del中断，信号始终只接收到2, 而16和17始终接收不到。
```

![image-20221113001302742](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221113001302742.png)

<center>
    Fig2-4. 接收中断信号才发生跳转
</center>

* 针对实验过程3，程序运行的结果是什么样子？时钟中断有什么不同？

这里设置时钟为1s， 在不使用del信号时，会经过5s的sleep再kill

![image-20221113001659222](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221113001659222.png)

<center>
    Fig2-5. 不使用del中断信号
</center>

而在1s之前使用ctrl c后，尽管收到了14信号值，但子进程们已经由2信号所中断

![image-20221113002001391](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221113002001391.png)

<center>
    Fig2-6. 1s之前使用ctrl c后
</center>

而在1s后5s内使用ctrl+c，**del信号会立即执行**而不用等待**sleep(5)**

![image-20221113002208081](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221113002208081.png)

<center>
    Fig2-7. 1s后5s内使用ctrl c后
</center>

* kill命令在程序中使用了几次？每次的作用是什么？执行后的现象是什么？

```
kill命令使用了两次，分别给子进程1和子进程2发送16，17信号，顺序不一定相同，但是子进程1先输出的概率大。通过将kill命令插入wait命令中间可以控制子进程执行顺序，因为父进程在第一个等待结束后才会发送下一个kill信号。

使用kill命令后，子进程接收到kill命令并调用stop函数，stop函数将wait_flag置为0，输出被杀死的信号并结束。
```

* 使用kill命令可以在进程的外部杀死进程。进程怎样能主动退出？这两种退出方式哪种更好一些？

```
进程调用return函数和exit函数都可以主动退出，而kill是强制退出。主动退出比较好，如果在某个子进程退出前父进程被kill强制退出，则子进程会被init进程接管；如果用kill命令杀死某个子进程而其父进程没有调用wait函数等待，则该子进程为处于僵死状态占用资源。
```



## 2.2 进程的管道通信

### 目的

编程实现进程的管道通信，通过观察、分析实验现象，深入理解进程管道通信的特点，掌握管道通信的同步和互斥机制。

### 实验内容

1）先猜想一下这个程序的运行结果。分析管道通信是怎样实现同步与互斥的；
2）然后按照注释里的要求把代码补充完整，运行程序；
3）修改程序并运行，体会互斥锁的作用，比较有锁和无锁程序的运行结果，并解释之。

### 实验结果

![image-20221113002842899](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221113002842899.png)

代码：

```
#include <unistd.h>
#include <signal.h>
#include <stdio.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <stdlib.h>
#include <string.h>
 
int pid1, pid2;
main()
{
    int fd[2];
    char InPipe[1000];
    //char c1 = '1', c2 = '2';
    char *w1=(char*)"1",*w2=(char*)"2";
    pipe(fd);
    while ((pid1 = fork()) == -1)
        ;
    if (pid1 == 0)
    {
        //printf("\n first process start \n");
        lockf(fd[1], 1, 0);//补充，锁定管道
         //补充，分200次向管道写入1
        for(int i=0;i<200;i++)
        write(fd[1],w1,1);
        sleep(3);
        lockf(fd[1], 0, 0);//补充，解除锁定
        exit(0);
    }
    else
    {
        while ((pid2 = fork()) == -1)
            ;
        if (pid2 == 0)
        {
           //printf("\n second process start \n");
          lockf(fd[1], 1, 0);
            //补充，分200次向管道写入2
            for(int i=0;i<200;i++)
            write(fd[1],w2,1);
            sleep(3);
            lockf(fd[1], 0, 0);
            exit(0);
        }
        else
        {
            wait(0);
            wait(0);
            //printf("\n parent process start \n");
            //补充，从管道中读出400个字符
            read(fd[0],InPipe,400);
            //加入字符串结束符
            InPipe[400]='\0';
            printf("%s\n", InPipe);
            exit(0);
        }
    }
}
```

实验结果：

![image-20221113003425890](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221113003425890.png)

<center>
    Fig2-8. 管道通信结果
</center>

* 你最初认为运行结果会怎么样？

```
屏幕会输出依次200个1和200个2.
```

* 实际的结果什么样？有什么特点？试对产生该现象的原因进行分析。

```
实际运行结果和猜想一致。如果进程1先执行则先输入1，然后进程2执行输入2
```

* 实验中管道通信是怎样实现同步与互斥的？如果不控制同步与互斥会发生什么后果？

```
通过使用函数lockf来锁住管道的写端口。如果不加锁会交错输出1和2
```

不加锁代码如下：

```
#include <unistd.h>
#include <signal.h>
#include <stdio.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <stdlib.h>
#include <string.h>
 
int pid1, pid2;
main()
{
    int fd[2];
    char InPipe[1000];
    //char c1 = '1', c2 = '2';
    char *w1=(char*)"1",*w2=(char*)"2";
    pipe(fd);
    while ((pid1 = fork()) == -1)
        ;
    if (pid1 == 0)
    {
        //printf("\n first process start \n");
        //lockf(fd[1], 1, 0);//补充，锁定管道
         //补充，分200次向管道写入1
          for(int i=0;i<200;i++)
           {
                write(fd[1],w1,1);
                sleep(0.1);
           }
       // lockf(fd[1], 0, 0);//补充，解除锁定
        exit(0);
    }
    else
    {
        while ((pid2 = fork()) == -1)
            ;
        if (pid2 == 0)
        {
           //printf("\n second process start \n");
           //lockf(fd[1], 1, 0);
            //补充，分200次向管道写入2
            for(int i=0;i<200;i++)
           {
                write(fd[1],w2,1);
                sleep(0.1);
           }
            //lockf(fd[1], 0, 0);
            exit(0);
        }
        else
        {
            wait(0);
            wait(0);
            //printf("\n parent process start \n");
            //补充，从管道中读出400个字符
            read(fd[0],InPipe,400);
            //加入字符串结束符
            InPipe[400]='\0';
            printf("%s\n", InPipe);
            exit(0);
        }
    }
}
```

![image-20221113005225188](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221113005225188.png)

<center>
    Fig2-9. 不加锁管道通信结果
</center>
## 2.3 页面置换

### 实验目的

模拟实现FIFO算法，LRU算法

### 算法思想

* FIFO
  1. 在分配内存页面数（AP）小于进程页面数（PP）时，当然是最先运行的AP个页面放入内存；
  2. 这时又需要处理新的页面，则将原来放的内存中的AP个页中最先进入的调出（FIFO），再将新页面放入；
  3. 以后如果再有新页面需要调入，则都按上述规则进行。

算法特点：所使用的内存页面构成一个队列。

***具体步骤：***

（1）初始化。设置两个数组page[ap]和pagecontrol[pp]分别表示进程页面数和内存分配的页面数，并产生一个随机数序列main[total_instruction]（这个序列由page[ap]的下标随机构成）表示待处理的进程页面顺序，diseffect置0。
（2）看main[]中是否有下一个元素，若有，就由main[]中获取该页面下标，并转（3），如果没有则转（7）。
（3）如果该页已在内存中，就转（2），否则转（4），同时未命中的diseffect加1。
（4）观察pagecontrol是否占满，如果占满则须将使用队列（在第（6）步中建立的）中最先进入的（就是队列的第一个单元）pagecontrol单元“清干净”，同时将page[]单元置为“不在内存中”。
（5）将该page[]与pagecontrol[]建立对应关系（可以改变pagecontrol[]的标志位，也可以采用指针链接，总之至少要使对应的pagecontrol单元包含两个信息：一是它被使用了，二是哪个page[]单元使用的。page[]单元也包含两个信息：对应的pagecontrol单元号和本page[]单元已在内存中）。
(6) 将用到的pagecontrol置入使用队列（这里的队列是一种FIFO的数据结构），返回（2）。
(7) 显示计算1-diseffect / total_instruction*100%，完成。

* LRU
  1. 当内存分配页面数（AP）小于进程页面数（PP）时，把最先执行的AP个页面放入内存。
  2. 当需调页面进入内存，而当前分配的内存页面全部不空闲时，选择将其中最长时间没有用到的那一页调出，以空出内存来放置新调入的页面（LRU）。

算法特点：每个页面都有属性来表示有多长时间未被CPU使用的信息。

***具体步骤：***

（1）初始化。设置两个数组page[ap]和pagecontrol[pp]分别表示进程页面数和内存分配的页面数，并产生一个随机数序列main[total_instruction]（这个序列由page[ap]的下标随机构成）表示待处理的进程页面顺序，diseffect置0。

（2）看序列main[]中是否有下一个元素，如果有，就由main[]中获取该页面下标，并转（3），如果没有则转（6）。

（3） 如果该page[]单元在内存中便改变页面属性，使它保留“最近使用”的信息，转（2），否则转（4），同时diseffect加1。

（4）看是否有空闲页面，如果有，就返回页面指针，并转到（5），否则，在内存页面中找出最长时间没有使用到的页面，将其“清干净”，并返回该页面指针。

（5）在需处理的page[]与（4）中得到的pagecontrol[]之间建立联系，同时让对应的page[]单元保存“最新使用”的信息，转（2）。

（6）如果序列处理完成，就输出计算1-diseffect / total_instruction*100%的结果，完成

代码如下：

```
#include<stdio.h>
#include<stdlib.h>
#define AP 10
#define PP 3
#define TOTAL_INSTRUCTION 20

int Queue[PP+1]={};
int head = 0;
int tail = 0;

int Stack[PP] = {};
int top = PP-1;
int bottom = PP-1;
//用于存储中间结果
int temp[PP][TOTAL_INSTRUCTION];
//检查pagecontrol是否还有空位
int isConEmpty(int* first_empty,int pagecontrol[],int control_num)
{
    int flag = 0;
    for(int iter = 0;iter<control_num;iter++)
    {
        if(pagecontrol[iter]==-1)
        {
            flag = 1;
            *first_empty = iter;
            break;
        }

    }
    return flag;
}
/*
* FIFO算法
* page[cur]是当前需要调入的页面
*/
void FIFO(int curpage,int page[],int pagecontrol[],int control_num)
{
    int first_empty = -1;
    //如果pagecontrol中有空位，则直接放入空位
    if(isConEmpty(&first_empty,pagecontrol,control_num))
    {
        pagecontrol[first_empty]=curpage;
        //该页面已在内存中
        page[curpage] = 1;
        //队列记录的是最先放入数据的位置而不是页面号
        Queue[tail]=first_empty;
        if((tail+1)%(PP+1) != head)
        {
            tail = (tail + 1)%(PP+1);
        }
        else
        {
            printf("队列已满！\n");
            exit(1);
        }
    }
    //如果没有空位，则替换最先
    else
    {
        page[pagecontrol[Queue[head]]]=0;
        pagecontrol[Queue[head]]=curpage;
        page[curpage] = 1;
        Queue[tail]=Queue[head];
        if((head+1)%(PP+1)!=tail)
        {
            head = (head+1)%(PP+1);
        }
        else
        {
            printf("队列空！\n");
            exit(1);
        }
        if((tail+1)%(PP+1) != head)
        {
            tail = (tail + 1)%(PP+1);
        }
        else
        {
            printf("队列已满！\n");
            exit(1);
        }
    }
}
//LRU算法
void LRU(int curpage,int page[],int pagecontrol[],int control_num)
{
    int first_empty = -1;
    //先检查在不在pagecontrol里面
    for(int iter = 0; iter<PP;iter++)
    {
        if(pagecontrol[iter]==curpage)
        {
            int iter2;
            for(iter2 = 0;iter2<PP;iter2++)
            {
                if(Stack[iter2]==iter)
                break;
            }
            for(;iter2<PP-1;iter2++)
            {
                Stack[iter2]=Stack[iter2+1];
            }
            Stack[top]=iter;
            return;
        }
    }
    //如果pagecontrol中有空位，则直接放入空位
    if(isConEmpty(&first_empty,pagecontrol,control_num))
    {
        pagecontrol[first_empty]=curpage;
        //该页面已在内存中
        page[curpage] = 1;
        //链表记录的是最先放入数据的位置而不是页面号

        for(int iter = 0;iter<PP-1;iter++)
        {
            Stack[iter]=Stack[iter+1];
        }
        bottom--;
        Stack[top]=first_empty;
    }
    //如果没有空位，替换最近使用最少
    else
    {
        page[pagecontrol[Stack[bottom+1]]]=0;
        pagecontrol[Stack[bottom+1]]=curpage;
        page[curpage] = 1;
        
        int tmp = Stack[bottom+1];
        for(int iter = 0;iter<PP-1;iter++)
        {
            Stack[iter]=Stack[iter+1];
        }
        
        Stack[top]=tmp;
    }
}
int main()
{
    //队列初始化为-1
    for(int iter=0;iter<PP+1;iter++)
    {
        Queue[iter]=-1;
    }
    //栈初始化为-1
    for(int iter=0;iter<PP;iter++)
    {
        Stack[iter]=-1;
    }
    /*定义变量
    * pagecontrol是内存分配的页面，初始化为-1
    */
    printf("-----------------------------------------------------\n");
    int page[AP];
    int pagecontrol[PP];
    for(int iter = 0;iter<PP;iter++)
    {
        pagecontrol[iter] = -1;
    }
    for(int iter = 0;iter<AP;iter++)
    {
        page[iter] = 0;
    }
    
    /*int pageseq[TOTAL_INSTRUCTION] = {};
    srand((unsigned int)getpid());
    for(int iter = 0;iter<TOTAL_INSTRUCTION;iter++)
    {
        pageseq[iter]=rand()%AP;
    }*/

    int pageseq[TOTAL_INSTRUCTION] = {7,0,1,2,0,3,0,4,2,3,0,3,2,1,2,0,1,7,0,1};
    printf("Number of page frame：%d\n",PP);
    printf("The sequence：\n%d",pageseq[0]);
    for(int iter = 1;iter<TOTAL_INSTRUCTION;iter++)
    {
        printf(" %d",pageseq[iter]);
    }
    printf("\n");
    int algorithm_type = 0;
    printf("Select the algorithm：0为FIFO，1为LRU。\n");
    scanf("%d",&algorithm_type);
    switch(algorithm_type)
    {
        case 0: {printf("FIFO:\n");break;}
        case 1: {printf("LRU:\n");break;}
        default: {printf("FAULT!!!\n");exit(1);}
    }
    //cur是当前页面序列下标
    int cur = 0;
    //diseffect是未命中次数
    int diseffect = 0;

    while(cur!=TOTAL_INSTRUCTION)
    {
        //如果序列当前页面在已经分配的页面中
        if(page[pageseq[cur]]==1)
            //FIFO什么都不用做
            //LRU要更新记录
        {
            if(algorithm_type==1)
            {
                LRU(pageseq[cur],page,pagecontrol,PP);
            }
        }
        //如果不在当前已经分配的页面中
        else
        {
            diseffect++;
            if(algorithm_type == 0)
            {
                FIFO(pageseq[cur],page,pagecontrol,PP);
            }
            else
            {
                LRU(pageseq[cur],page,pagecontrol,PP);
            }
        }
        
        //记录pagecontrol的值便于输出
        for(int iter = 0;iter<PP;iter++)
        {
            temp[iter][cur]=pagecontrol[iter];
        }     
        //查看下一个待调用的页面
        cur++;
    }
    double hit_rate = 100*(1-((double)diseffect/TOTAL_INSTRUCTION));
    printf("-----------------------------------------------------\n");
    printf("Visualization(-1表示空)：\n");
    for(int iter = 0; iter<PP;iter++)
    {
        printf("%d",temp[iter][0]);
        for(int iter2 = 1;iter2<TOTAL_INSTRUCTION;iter2++)
        {
            printf("\t%d",temp[iter][iter2]);
        }
        printf("\n");
    }
    printf("-----------------------------------------------------\n");
    printf("TOTAL_INSTRUCTION：%d\n",TOTAL_INSTRUCTION);
    printf("diseffect：%d\n",diseffect);
    printf("1-diseffect / total_instruction*100%%：%.2f%%\n",hit_rate);
    printf("-----------------------------------------------------\n");
    return 0;
}

```

### 实验结果

实验参数：

```
#define AP 10
#define PP 3
#define TOTAL_INSTRUCTION 20
```

为方便观察，固定访问序列为

```
7, 0, 1, 2, 0, 3, 0, 4, 2, 3, 0, 3, 2, 1, 2, 0, 1, 7, 0, 1
```

**FIFO结果**：

FIFO采用队列数据结构，在实现上最简单

![image-20221113114631058](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221113114631058.png)

<center>
    Fig2-10. FIFO结果
</center>

为进一步观察Belady现象，

设置访问12个序列`0,1,5,3,0,1,4,0,1,5,3,4`, 分别检验page frame为3 和4 的情况

![image-20221113123012454](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221113123012454.png)

<center>
    Fig2-11. Belady现象
</center>

可以看到增加了page frame容量后，命中率不增反减。

解释：

> 实页数增加 —> 能贮存的页数增加 —> 哪些页？—> 后面来的页
> 先进先出的替换算法，完全不考虑使用频率，即使增加了实页数，多贮存的部分接下来常访问可能性也不一定大（看运气），也就并不一定能增加命中率。
>
> 在计算机存储中，贝拉迪异常是一种现象，其中**增加页面帧数导致某些内存访问模式的页面错误数增加**。使用先进先出（FIFO）的页面替换算法时通常会遇到这种现象。在FIFO中，页面错误可能会随着页面框架的增加而增加，也可能不会增加，但是在优化和基于堆栈的算法（如LRU）中，随着页面框架的增加，页面错误会减少，LászlóBélády在1969年证明了这一点。
>
> reference: https://zhuanlan.zhihu.com/p/141096119



**LRU结果**

LRU采用堆栈数据结构，在实现上需要iter来标志使用频率

![image-20221113123654873](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221113123654873.png)

<center>
    Fig2-11. LRU结果
</center>

可以看到

```
2  2  2             2  2  2
0  0  0             0  0  3
1  1  3(LRU)        1  1  1 (FIFO)
```

LRU将`1->3`而不是`0->3`， 因为1最久没有被访问了

![image-20221113125103078](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221113125103078.png)

<center>
    Fig2-11. LRU过程，+表示iter
</center>
