---
{"dg-publish":true,"permalink":"//jmm-happens-before/","tags":"gardenEntry","dgHomeLink":true,"dgPassFrontmatter":false}
---


>[!abstract] 堆和栈
>

![](https://tva1.sinaimg.cn/large/e6c9d24egy1h602y75au1j20wo0ls3zf.jpg)

共享的变量一定在堆空间中。

>[!abstract] 硬件层面的内存模型

可以看到线程栈其实也是在这个内存里面，但是线程栈里面的变量是线程独有了，多线程的问题大多数是共享变量引起的，也 就是堆里面的变量

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h608190o38j218i0s4q4q.jpg)

CPU是一个运算器，他需要处理的数据还是需要从其他地方拿。而与这些地方交互就需要等待，
而通常，CPU的速度远远大于内存的读取速度，所以在这里等待内存读取数据，是CPU资源的一种浪费，所以进而有了寄存器，和缓存。
而CPU运算会从内存中读取数据到缓存里面，然后从缓存读取数据到寄存器供给CPU计算。CPU计算完后会把新值从寄存器放回到缓存中，然后在某个时间点再从缓存里面刷回到内存里面。比如当CPU需要在缓存中存其他东西的时候，那些用不到的值就要刷回主存里面去。通常缓存的更新单位是“缓存行”，也就是一次性会有一个或多个缓存行读取数据进来，或者从缓存行刷数据回内存。这个缓存行也就是GC里面的卡表的伪共享问题提到的。

现代计算机一般都有2个以上cpu，这样就使得同时有多个线程一起工作，
但同样也会导致了一些问题。因为缓存的存在，所以会导致数据不一致的问题。
一般是由缓存一致性协议来解决 

最主要是多个线程，在写-写，和读-写场景下的，竞争和写可见的问题。

写，写场景比如
```java

public static int count = 0;  
public static void main(String[] args) {  
    Runnable runnable = new Runnable(){  
        @Override  
        public void run(){  
            count+=1;  
        }  
    };  
    new Thread(runnable).start();  
    new Thread(runnable).start();
    Thread.sleep(10);  
  
	System.out.println(count);
}
```

这个结果可能不是2，是1，因为两个线程是同时进行的，线程A，B拿到的都是旧值，然后各自加1然后再刷回去，

读，写场景
以上代码，即使两个线程是顺序执行的，那么在线程A还 没有将count+1的结果刷回内存，那么线程B从内存拿出来的还是旧值

>[!abstract] 编程语言层面的内存模型-抽象

![](https://tva1.sinaimg.cn/large/e6c9d24egy1h604l1e4q9j21460u0q3y.jpg)

java在语言层面上也抽象出了一个内存模型，**因为java是跨平台的语言**，它希望能屏蔽底下硬件和操作系统的访问差异，就是提出各种JMM规则来达到各种平台下内存访问一致的效果，然后底下硬件和操作系统怎么实现这个JMM规则java程序猿不需要关心了。

缓存的出现--> 抽象为工作内存 --> 可见性问题
指令重排序优化--> 有序性的问题

**禁止缓存和编译优化，显然有点极端，所以我们要按需禁用缓存，以及按需编译优化
这样的约束规则，就是JMM内存模型要解决的问题。**

因为这个JMM规则，java编译器会去帮我做一些事情来达到内存访问一致。

>[!abstract] 重排序


 CPU为了提高效率，可以通过指令流水线并发的执行两条指令。但是前提是没有前后相关性
 
![](https://tva1.sinaimg.cn/large/e6c9d24egy1h60djgqk6tj21gi0ty75g.jpg)

CPU会寻找那些可以并发执行的指令，在CPU层面做重排序优化

但是为了给CPU减负，java虚拟机也会在编译层面去做优化，所以在代码被编译器翻译成字节码，或者字节码通过jit编译器编译成汇编指令，都已经做了优化了

所以重排序的出现，就是为了让cpu能更快的执行程序而诞生的。

**在单线程背景下其实是没有问题的，但是在多线程的背景下就会出现问题**
时间片切换

>[!abstract] happen before

^53c5aa

[[概念/Happens-before原则#^f02cdf|Happens-before原则#^f02cdf]]

保证和不保证的关系

[[问题Done/JMM/重排序和happens-before和原子性有序性可见性之间的关系|重排序和happens-before和原子性有序性可见性之间的关系]]

八个规则。可以利用来解决多线程环境下的有序性，可见性问题
1. 程序次序规则
2. 管程锁定规则
3. volatile变量规则
4. 线程启动规则
5. 线程终止规则
6. 线程中断规则
7. 对象终结规则
8. 传递性

以上8个规则是自然而然的happens-before关系

>[!abstract] volatile

# 一 · 解决可见性


```java
  
public class Consumer implements Runnable {  
    private Box box;  
  
    public Consumer(Box box) {  
        this.box = box;  
    }  
  
    @Override  
    public void run() {  
        while(true) {  
            if(!box.isFoodEmpty){  
                Food food = box.getFood();  
                box.addGetCount();  
                box.isFoodEmpty= true; 
                //System.out.println("consumer get :"+food);  
            }  
        }    }}
```

```java
public class Producer implements Runnable {  
    private Box box;  
    private Food[] foods = new Food[]{new Food("hamberger",23.4),new Food("pizza",12),new Food("apple",3)};  
    private Random random = new Random();  
    public Producer(Box box){  
        this.box = box;  
    }  
  
  
    @Override  
    public void run() {  
        while(true) {  
            if(box.isFoodEmpty) {  
                int i = random.nextInt(3);  
                box.setFood(foods[i]);  
                box.addPutCount();  
                box.isFoodEmpty=false;  
                //System.out.println("producer put: "+foods[i]);  
            }  
        }    }}
```

如果这里的`isFoodEmpty`不是volatile，那么会有之前所述的的我更改的数据，可能没有及时的刷进内存，那么Consumer或者是Producer的更改，没有被对方看见，此时，大家都在那里空转。

实验可以看见

```java
//Box
public volatile boolean isFoodEmpty = true;
```

>[!note] 现在把这个变量申明为volatile，那么他会使得缓存失效，
如果是对这个变量写操作，那么是强制性的写回主内存
如果是对这个变量读操作，那么是强制性的从主内存里面去读取。

那么两个线程的一读一写，就解决了上面的这个问题。就是可见性的问题

这里volatile是**保证**了我写的数据，一定会被其他线程看到。没有这个volatile，可能也会看到，但是不保证，在高并发的场景下，根据墨菲定律，那么一定会发生bug

>[!note] 其他所有的对于该线程可见的变量也一并写回内存，已经从内存里读

```java
nonVolatileA = 1;
nonVolatileB = 2;
VolatileC = 3;
nonVolatileD = 4;
```

其实都会写回，对于线程可见所有变量
A,B,C写回，只是这里到C这里就触发强制写回了，D还有没更新

```java
a = nonVolatileA;
b= VolatileB;
C= nonVolatileC;
```

b,c立即从内存里面读取，a是因为赋值了，然后b这里重新触发读，读回来的A可能不一样，但是a已经赋值了，只能下一轮了。


>[!note] 这里可见性和happens-before的关系
指的是，生产者线程如果比消费者线程先执行的话，那么它给`isFoodEmpty`赋值的操作，就是`isFoodEmpty`的值是一定可以被消费者线程所看到的。


# 二 · 解决有序性的问题

上面及时用`volatile`修饰了`isFoodEmpty`，还会存在一个问题就是
```java
            if(!box.isFoodEmpty){  
                Food food = box.getFood();  
                box.addGetCount();  
                box.isFoodEmpty= true; 
                //System.out.println("consumer get :"+food);  
            }  
```

因为`Food food = box.getFood()`这条语句，和`box.isFoodEmpty`，他们没有数据依赖性，所以他们有可能被重排序，比如变成

```java
if (box.isFoodEmpty) {  
    box.isFoodEmpty = false;  
    int i = random.nextInt(3);  
    box.setFood(foods[i]);  
    box.addPutCount();  
  
  
    System.out.println("producer put: " + foods[i]);
```

因为`isFoodEmpty=false`这里出发了强制写，后面两个指令并不能**保证**及时写回内存
生产者先把标志位写回去了，但是实际上还没有生产，然后消费者检测到标志位已经变了，
所以消费者拿到的food有可能是null。从而引发错误

实验可以看见

![](https://tva1.sinaimg.cn/large/e6c9d24egy1h60js2uolxj20tg08ijsl.jpg)



>[!note] volatile会通过内存屏障禁止重排序

```java
                Food food = box.getFood();  //1
                box.addGetCount();   //2
                box.isFoodEmpty= true;  //3
```

`isFoodEmpty`是volatile，那么前面两个语句是不能排到语句3后面的
当然1，2他们还是可以重排序的，

```java
            if(box.isFoodEmpty) {  //1
                int i = random.nextInt(3);  
                box.setFood(foods[i]);  
                box.addPutCount();  
```

读操作的话，以上例子，任何操作是不能重排序到语句1前面的。

>[!note] 字节码指令的重排序

因为上面例子简单，其实真正不能重排序指的是字节码指令，然后这个约束会传递给下面的汇编指令。

什么意思呢，就是比如 ,volatile `instance = new Object()`
那么其实这里的步骤是分为
1. 分配内存
2. 调用构造函数，初始化实例变量
3. 引用赋给局部变量表。

真正改变的的那一步是在第三步，所以是在第三步后面插入一个内存屏障，那么前面的指令是不能够排序到第三步后面的

如果instance没有用volatile修饰，那么第3步和第2步有可能会重排序，那么其他线程用到的可能是一个没有初始化的一个对象实例

[[程序分析/DCL例子|DCL例子]]


#  三 · 解决原子性

上面的消费者生产者的例子

![](https://tva1.sinaimg.cn/large/e6c9d24egy1h60l5qrfroj20xa0gs77j.jpg)

不合理，
出现的原因是因为在打印语句执行的时候，跳转到其他线程执行去了。

volatile无法解决原子性


所以volatile称为是最轻量级的同步机制，要解决原子性需要其他如synchronized

>[!note] volatile实现原理

[[永乐大典-并发/volatile实现原理|volatile实现原理]]

>[!abstract] synchronized

# 一 · 解决可见性

代码块出来的时候，他就会将线程可见的所有变量刷回到主内存里面去
而在进入这个代码块的时候，他会强制讲线程可见的所有变量重新从内存里面读取

# 二 · 解决有序性

对共享变量的写

```java
//
a=1; //1
synchronized{ //2
	b=1;
	c=2;
} //3
d=3;//4
```

a,b,c不能排到 3后面去，d不能排到2前面去
如果重排到3后面去了，那么就会失去**保证**，保证变量都及时的刷回内存
代码段里面的还是可以根据数据无依赖性重排序的


```java
//1
synchronized{//2
	x = a;
	y = b;
}
z = c;//3
```

x,y,z处的指令不能重排序到1这里。
如果重排了，那就失去**保证**，保证变量都重新从内存里面读取。


# 三 · 解决原子性问题

原子性问题的源头在于线程切换,所以理论上禁止线程切换就能解决这个问题

互斥操作，[[永乐大典-并发/synchronized原理及锁升级|synchronized原理及锁升级]]

>[!example] System.out.println

[[问题Done/JMM/看程序，加了一个语句就不一样了|看程序，加了一个语句就不一样了]]

