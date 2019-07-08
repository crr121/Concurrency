[TOC]

## 信号量机制

### 什么是信号量

- 信号量是一种变量或抽象数据类型，用于控制并发系统中多个进程对公共资源的访问
- 一个普通的信号量是一个普通的变量，可以对它进行递增或递减，或切换等操作。

引用一波百度百科的解释：

以一个停车场的运作为例。简单起见，假设停车场只有三个车位，一开始三个车位都是空的。这时如果同时来了五辆车，看门人允许其中三辆直接进入，然后放下车拦，剩下的车则必须在入口等待，此后来的车也都不得不在入口处等待。这时，有一辆车离开停车场，看门人得知后，打开车拦，放入外面的一辆进去，如果又离开两辆，则又可以放入两辆，如此往复。
在这个停车场系统中，车位是公共资源，每辆车好比一个线程，**看门人**起的就是信号量的作用。

### 信号量的分类

- 计数信号量：信号量是一个任意的整数
- 二值信号量：只有二进制的0或1

### 信号量的工作机制

信号量的值表示用于控制并发系统中多个进程对公共资源的访问，只有P/V操作能够修改信号量的值

- P（pass）操作：

  - 完成一次wait()，信号量值减一，如果新的信号量值大于0，则进入临界区

  - 如果新的信号量的值小于0，那么该进程将被阻塞（被添加到semaphor的等待进程队列里面waiting queue）

- V（release）操作

  - 完成一次signal()，信号量值加一，如果递增之前，semaphore的值小于0（表示有进程正在等待获取资源），从waiting queue中挑选一个进程到ready queue里面

#### 实战

例：若有一售票厅只能容纳300人，当少于300人时，可以进入；否则，需在外等候。若将每一个购票者作为一个进程，请用P（wait）、V（signal）操作编程，并写出信号量的初值。（强调：只有一个购票窗口，每次只能为一位购票者服务）

#### 分析：

1）首先确定临界区资源：售票厅，购票窗口

2）初始化信号量：

- 售票厅的信号量可以用一个计数信号量emptyCount，初始化fullCount= 0（fullCount<= 300）
- 购票窗口的只有一个，只有两种状态：占用和空闲，用一个二值信号量IsBusy, 初始化isBusy = 1 为空闲



    user process {
                    P(emptyCount)
                    
                    进入大厅
                    
                    P(IsBusy)
    
                    窗口购票
    
                    V(IsBusy)
                    
                    退出购票窗口
                    
                    V(emptyCount)
                    
                    退出大厅

### 信号量的特点

- 适用于控制一个仅支持有限个用户的共享资源
- 与锁的区别：Lock和synchronized是锁的互斥，一个线程如果锁定了一资源，那么其它线程只能等待资源的释放。也就是一次只有一个线程执行，这到这个线程执行完毕或者unlock；而Semaphore可以控制多个线程同时对某个资源的访问。        
- semaphore锁的是什么：信号量用在多线程多任务同步的，一个线程完成了某一个动作就通过信号量告诉别的线程，别的线程再进行某些动作。也就是说Semaphore不一定是锁定某个资源，而是流程上的概念。
  - 比方说生产者消费者模型中，当共享缓冲区中已经满了，那么producer不能继续生产处于阻塞状态（此时缓冲区的信号量为emptyCount = 0），这个时候消费者一旦消费了一个数据之后（emptyCount = 1），那么又可以通知生产者继续生产,此时正处于阻塞状态的producer重新获取到信号量，从waiting queue 进入到ready queue。
  - 此时对于生产者消费者模型来说，这里的信号量并没有锁共享资源，而只是起到了一个通知作用，或者从另一个角度来说，相当于一个触发器，当达到了某一条件时，触发另外一个事件的发生

### Java中如何使用信号量

#### 构造函数

创建一个固定许可数量的信号量（非公平：线程获得许可的顺序是不定的）

```java
public Semaphore(int permits) {    
    sync = new NonfairSync(permits);
}
```

创建一个固定许可数量的信号量（公平：先后顺序分配许可FIFO）

```java
  public Semaphore(int permits, boolean fair) {
        sync = fair ? new FairSync(permits) : new NonfairSync(permits);
    }
```

非公平分配方式比公平分配方式更加高效：

在恢复一个被挂起的线程与该线程真正运行之间存在着严重的延迟。假设线程A持有一个锁，并且线程B请求这个锁。由于锁被A持有，因此B将被挂起。当A释放锁时，B将被唤醒，因此B会再次尝试获取这个锁。与此同时，如果线程C也请求这个锁，那么C很可能会在B被完全唤醒之前获得、使用以及释放这个锁，B获得锁的时刻并没有推迟，C更早的获得了锁，并且吞吐量也提高了。

#### 应用

```java
package concur;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Semaphore;

public class Semaphoer {
    public static void main(String[] args){

        ExecutorService executor = Executors.newCachedThreadPool();
        //临界区的资源数为5
        Semaphore semaphore = new Semaphore(5);
        //创建10个线程获取临界区的资源
        for(int i = 0; i < 10; i++){
        
           Runnable runnable =  new Runnable(){
               @Override
               public void run() {
                   try {
                       //获取许可
                       semaphore.acquire();   
                       
                       for(int i = 0; i < 99990; i++);
                       //释放许可
                       semaphore.release(); 
                       
                   } catch (InterruptedException e) {
                       e.printStackTrace();
                   }
               }
           };
           
            executor.execute(runnable);
        }

        executor.shutdown();
    }
}

```

#### 生产者消费者模型

生产者消费者模型中的共享资源是一个固定大小的缓冲区，该模式需要当缓冲区满的时候，生产者不再生产数据，直到消费者消费了一个数据之后，才继续生产；同理当缓冲区空的时候，消费者不再消费数据，直到生产者生产了一个数据之后，才继续消费

如果要通过信号量来解决这个问题：关键在于找到能够跟踪缓冲区的size大小变化，并根据缓冲区的数量变化来控制消费者和生产者线程之间的协作和运行

那么很容易很够想到用两个信号量：empytyCount和fullCount分别来表示缓冲区满或者空的状态，进而能够更加容易控制消费者和生产者到底什么时候处于阻塞状态，什么时候处于运行状态

- emptyCount = N ; fullCount = 0 ; useQueue = 1

同时为了使得程序更加具有健壮性，我们还添加一个二进制信号量useQueue,确保队列的状态的完整性不受损害。例如当两个生产者同时向空队列添加数据时，从而破坏了队列内部的状态，使得其他计数信号量或者返回的缓冲区的size大小不具有一致性。（当然这里也可以使用mutex来代替二进制信号量）

生产者：

```
produce:
    P(emptyCount)//信号量emptyCount减一
    P(useQueue)//二值信号量useQueue减一，变为0（其他线程不能进入缓冲区，阻塞状态）
    putItemIntoQueue(item)//执行put操作
    V(useQueue)//二值信号量useQueue加一，变为1（其他线程可以进入缓冲区）
    V(fullCount)//信号量fullCount加一
```



```
consume:
    P(fullCount)//fullCount -= 1
    P(useQueue)//useQueue -= 1(useQueue = 0)
    item ← getItemFromQueue()
    V(useQueue)//useQueue += 1 (useQueue = 1)
    V(emptyCount)//emptyCount += 1
```



#### 死锁恢复

使用信号量解决死锁问题**deadlock recovery**

##### 我们构建这样一个死锁场景：

假设你和我想要享受饼干和牛奶。有一盒饼干，一加仑牛奶，我们每个人都有一个盛牛奶的杯子和盛饼干的盘子。
我们抛硬币决定谁先选(我选正面，你选反面)。我们每个人都有30秒的时间去拿饼干或倒牛奶，否则下一个人就开始选了。另一条规则是，一旦您决定抓取一个项目(牛奶或饼干)，您必须继续这样做，直到项目可用(或时间结束)。

既然我掷硬币赢了，我就先选。牛奶和饼干都有，所以我拿起牛奶倒了一杯。同时我也留着牛奶，尽管我不再需要它了。现在轮到你了。你拿起一袋饼干(这时你没有选择，只有饼干了，没有牛奶)，把一些放在你的盘子里。并且你也留着那袋饼干。现在轮到我了。我想吃饼干，但饼干还在你那。我的30秒到此为止，该你了。你去拿牛奶，但牛奶仍然在我这。你的30秒到此为止，轮到我了。而我继续等着饼干，我的时间到了。循环往复，我们陷入了僵局。

要修复死锁(除了一些对死锁检测的支持之外)，我们每个人都需要修改我们的操作——在我倒了一杯牛奶之后，我把加仑牛奶放回去，这样你就可以拿了。吃完一盘饼干后，你放回那袋饼干，这样我就能拿了。

- 这里我们将共享资源饼干和牛奶看成信号量，令信号量A ==加仑牛奶，信号量B =一包饼干

```
Me: P(A)
You: P(B)
Me: P(B) (I can't do it because you have B, so I wait forever)
You: P(A) (you can't do it because I have A, so you wait forever).
```

为了预防死锁，我们做如下修改

```
Me: P(A) then V(A) (grab the milk, pour a glass, put down the milk)
You: P(B) then V(B) (grab the cookies, put some on your plate, put down the cookies)
Me: P(B) then V(B)  (grab the cookies, put some on my plate, put down the cookies)
You: P(A) then V(A) (grab the milk, pour yourself a glass, put down the milk)
```

##### 实现

```java
package concur;

import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

public class WorkThread1 {
    private Semaphore semaphore1,semaphore2;
    public WorkThread1(Semaphore semaphore1,Semaphore semaphore2){
        this.semaphore1=semaphore1;
        this.semaphore2=semaphore2;
    }
    public void run() {
        try {
            semaphore2.acquire();//先获取Semaphore2
            System.out.println(Thread.currentThread().getId()+" 获得Semaphore2");
            TimeUnit.SECONDS.sleep(5);//等待5秒，让WorkThread1先获得Semaphore1
            semaphore1.acquire();//获取Semaphore1
            System.out.println(Thread.currentThread().getId()+" 获得Semaphore1================");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

```

```java
package concur;

import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

class WorkThread2 extends Thread{
    private Semaphore semaphore1,semaphore2;

    public WorkThread2(Semaphore semaphore1,Semaphore semaphore2){
        this.semaphore1=semaphore1;
        this.semaphore2=semaphore2;
    }
    public void releaseSemaphore2(){
        System.out.println(Thread.currentThread().getId()+" 释放Semaphore2");
        semaphore2.release();
    }
    public void run() {
        try {
            semaphore1.acquire(); //先获取Semaphore1
            System.out.println(Thread.currentThread().getId()+" 获得Semaphore1");
            TimeUnit.SECONDS.sleep(5); //等待5秒让WorkThread1先获得Semaphore2
            semaphore2.acquire();//获取Semaphore2
            System.out.println(Thread.currentThread().getId()+" 获得Semaphore2=============");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}


```

```java
package concur;

import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

public class SemphoreTest {
    public static void main(String[] args) throws InterruptedException {
        Semaphore semaphore1=new Semaphore(1);
        Semaphore semaphore2=new Semaphore(1);
        new WorkThread1(semaphore1, semaphore2).start();
        new WorkThread2(semaphore1, semaphore2).start();
    }
}

```

```
12 获得Semaphore2
13 获得Semaphore1
```

接下来我们每次获得信号量之后都进行释放，那么就能解决死锁的问题

```java
package concur;

import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

public class WorkThread1 extends Thread{
    private Semaphore semaphore1,semaphore2;
    public WorkThread1(Semaphore semaphore1,Semaphore semaphore2){
        this.semaphore1=semaphore1;
        this.semaphore2=semaphore2;
    }
    public void run() {
        try {
            semaphore2.acquire();//先获取Semaphore2     
            System.out.println(Thread.currentThread().getId()+" 获得Semaphore2");
             semaphore2.release();//获取之后释放
            TimeUnit.SECONDS.sleep(5);//等待5秒，让WorkThread1先获得Semaphore1
            semaphore1.acquire();//获取Semaphore1       
            System.out.println(Thread.currentThread().getId()+" 获得Semaphore1================");
             semaphore1.release();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

```

```java
package concur;

import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

class WorkThread2 extends Thread{
    private Semaphore semaphore1,semaphore2;

    public WorkThread2(Semaphore semaphore1,Semaphore semaphore2){
        this.semaphore1=semaphore1;
        this.semaphore2=semaphore2;
    }

    public void run() {
        try {
            semaphore1.acquire(); //先获取Semaphore1
          
            System.out.println(Thread.currentThread().getId()+" 获得Semaphore1");
              semaphore1.release();
            TimeUnit.SECONDS.sleep(5); //等待5秒让WorkThread1先获得Semaphore2
            semaphore2.acquire();//获取Semaphore2
           
            System.out.println(Thread.currentThread().getId()+" 获得Semaphore2=============");
             semaphore2.release();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}


```



```java
12 获得Semaphore2
13 获得Semaphore1
13 获得Semaphore2=============
12 获得Semaphore1================
```

