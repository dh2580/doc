## Java多线程之`synchronized`
多线程环境下，涉及到对于临界区代码的并发访问，有多种同步方位策略，其中`synchronized`作为最原始的一种同步访问策略存在已久。相比于`API`层面的`Lock`锁，`synchronized`是基于指令的语义层面的一种同步机制，通过在字节码中插入`monitorenter`、`monitorenter`指令来实现，相关使用以及具体实现会在下面具体阐述。
#### `synchronized` 三种使用场景
+ 锁定对象，对一个对象进行加锁
```java
public class Console {

    public static void main(String[] args) {

        Console console = new Console();

        synchronized (console) {

            System.out.println("hello, world");

        }
    }

}
```
锁定一个具体的实例对象，在锁定的代码块中，同一时间，最多只有一个线程能进来执行，其他线程在代码块外阻塞等待

+ 锁定实例方法，对具体的实例方法加锁
```java
public class Console {

    public synchronized void doSomeThing() {

        System.out.println("hello,wolrd");

    }

}
```
锁定一个具体的实例方法（其实锁定的是当前对象），同一时间，最多只有一个线程能进来执行，其他线程方法外阻塞等待

+ 锁定静态方法，对静态方法加锁
```java
public class Console {

    public synchronized static void doSomeThing() {

        System.out.println("hello,wolrd");

    }

}
```
与锁定实例方法不同，锁定静态方法是对当前类的`class`对象加锁，同样同一时间，最多只有一个线程能进来执行，其他线程方法外阻塞等待，和如下示例等同
```java
public class Console {

    public static void doSomeThing() {

        synchronized(Console.class) {
            System.out.println("hello,wolrd");
        }
    }

}
```

#### `synchronized` 与 `wait/notify`机制
+ 当一个线程持进入到`synchronized`修饰的代码块，即表示该线程持有`锁`，在执行完同步代码块之前，其他线程只能在同步代码块外阻塞等待，并且这个等待是无限等待。

+ 如果这个线程在持有锁期间，想要释放当前持有的锁，可以借助`wait/notify`机制实现。在锁定代码块中，通过调用锁定对象的`wait()`方法，当前线程会释放所持有的锁，这样其他线程能够获取到锁。

+ 当线程在锁定代码中调用`wait()`方法后，线程会进入`WAITING`状态，等待唤醒，如果需要唤醒该线程，则需要其他线程调用锁定对象的`notify()`或者`notifyAll()`。

具体代码如下所示
```java
public class Console {

    public static void main(String[] args) throws InterruptedException {
        Console console = new Console();

        Thread t1 = new Thread(() -> {
            synchronized (console) {

                try {

                    System.out.println("start......");
                    console.wait();
                    System.out.println("end......");

                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

            }
        });
        t1.start();

        TimeUnit.SECONDS.sleep(1);

        Thread t2 = new Thread(() -> {

            synchronized (console) {

                System.out.println("new start......");
                console.notify();
                System.out.println("new end......");

            }

        });
        t2.start();
    }

}
```
输入如下
```
start......
new start......
new end......
end......

Process finished with exit code 0
```
备注：在调用锁定对象`wait()`方法时，需要确保该线程已经持有锁，更确切的来说是必须在同步代码块内调用`wait()`方法，否则会抛出`java.lang.IllegalMonitorStateException`异常，如下代码所示
```java
public class Console {

    public static void main(String[] args) throws InterruptedException {
        Console console = new Console();
        console.wait();
    }

}
```
输入如下
```java
Exception in thread "main" java.lang.IllegalMonitorStateException
	at java.lang.Object.wait(Native Method)
	at java.lang.Object.wait(Object.java:502)
	at Console.main(Console.java:9)

Process finished with exit code 1
```

*释放当前锁不仅可以通过调用`wait()`方法释放，也可以通过*`wait(long timeout)`方法释放，只不过`wait(long timeout)`方法不同的是，线程调用该方法后，进入`TIMED_WAITING`（有限等待）状态，如果等待超时则退出`TIMED_WAITING`（有限等待）状态，重新抢锁，如果抢锁失败，则进入`BLOCKED`（阻塞）状态，关于线程的状态变化可参考下图
![线程状态转换](images/线程状态转换.png)

#### `synchronized` 底层实现
在字节码层面，`synchronized`经过编译后，会在同步块前后分别形成`monitorenter`、`monitorexit`两个字节码指令，这两个字节码指令将同步代码块包围起来，如下所示Java代码
```java
public void test() {

    synchronized (this) {

        System.out.println();

    }
}
```
经过编译后的字节码如下所示

```
public test()V
    TRYCATCHBLOCK L0 L1 L2 null
    TRYCATCHBLOCK L2 L3 L2 null
   L4
    LINENUMBER 9 L4
    ALOAD 0
    DUP
    ASTORE 1
    MONITORENTER            //表示同步代码块的开始
   L0
    LINENUMBER 11 L0
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
    INVOKEVIRTUAL java/io/PrintStream.println ()V
   L5
    LINENUMBER 13 L5
    ALOAD 1
    MONITOREXIT             //表示同步代码块的结束
   L1
    GOTO L6
   L2
   FRAME FULL [Console java/lang/Object] [java/lang/Throwable]
    ASTORE 2
    ALOAD 1
    MONITOREXIT             //如果同步代码块执行中抛异常，也会结束
   L3
    ALOAD 2
    ATHROW
   L6
    LINENUMBER 14 L6
   FRAME CHOP 1
    RETURN
   L7
    LOCALVARIABLE this LConsole; L4 L7 0
    MAXSTACK = 2
    MAXLOCALS = 3
}
```
此外，`monitorenter`、`monitorexit`这两个字节码指令都需要一个`reference`类型的参数来指明要锁定和解锁的对象，也就是我们传递进入的锁定对象。如果`java`程序中的`synchronized`明确指定了对象参数，那就是这个对象的`reference`，如果没有明确指定，那就根据`synchronized`修饰的是实例方法还是静态方法，去取对应的对象实例或`class`对象来作为锁定对象。（*摘自<<深入理解Java虚拟机>>*）

上面只是`synchronized`关键词在字节码层面的实现，实际在JVM内部执行时，每一锁定对象都有一个`ObjectMonitor`对象在JVM底层与之对应，关于`ObjectMonitor`对象的内部数据结构如下所示：
```c
ObjectMonitor() {
    _header       = NULL;//markOop 对象头
    _count        = 0;
    _waiters      = 0,// 等待线程数
    _recursions   = 0;// 重入次数
    _object       = NULL;// 监视器锁寄生的对象. 锁不是平白出现的, 而是寄托存储于对象中.
    _owner        = NULL;// 指向获得 ObjectMonitor 对象的线程或基础锁
    _WaitSet      = NULL;// 处于 wait 状态的线程, 会被加入到 waitSet;
    _WaitSetLock  = 0;
    _Responsible  = NULL;
    _succ         = NULL;
    _cxq          = NULL;
    FreeNext      = NULL;
    _EntryList    = NULL;// 处于等待锁 block 状态的线程, 会被加入到 entryList;
    _SpinFreq     = 0;
    _SpinClock    = 0;
    OwnerIsThread = 0;
    _previous_owner_tid = 0;// 监视器前一个拥有者线程的 ID
}
```
*因为JVM底层使用C++来实现的，所以上述是C++代码*

如上述结构所示，其中，
+ `_object`指向了当前锁定对象，如果`synchronized`有指定对象实例，则该`_object`就指向该对象实例，如果`synchronized`修饰的是实例方法或者静态方法，则`_object`指向的就是实例对象或者`class`对象。
+ `_owner`指向了获得锁的线程

+ `_recursions`表示当前持有锁的线程的重入次数，因为`synchronized`是可重入的，所以每进入一次同步代码块，该属性就会+1，每退出一次同步代码块，该属性就会-1，当线程释放锁时，该属性的值为0

+ `_WaitSet`表示处于等待状态（`WAITING`、`TIMED_WAITING`）的线程集合，是一个`Set`结构

+ `_EntryList`表示处于阻塞状态（`BLOCKING`）的线程集合，是一个`Set`结构

`_EntryList`、`_WaitSet`、`_owner`之间的转换如下所示
![线程状态转换](images/entrylist_waitset_owner转换.jpeg)

如上图所示，有两个集合`_EntryList`、`_WaitSet`，用保存等待获取锁的线程集合，`_owner`指向当前持有锁的线程。
+ 当多个线程同时访问同步代码块时，如果当前锁未被任何线程持有，则会尝试获取锁，如果获取成功，则会将`_owner`指向自己，如果获取失败，则进入`_EntryList`集合，进入`BLOCKING`状态。如果当前锁已经被其他线程持有，则直接进`_EntryList`集合，进入`BLOCKING`状态。
+ 如果在持有的线程在执行期间，调用了`wait()`或者`wait(long timeout)`方法，则会释放锁，对`_owner`属性进行复位，然后进入`_WaitSet`集合，等待被唤醒或者等待超时。

+ 如果`_WaitSet`集合中的线程等待超时，则会在超时之后尝试获取锁，如果获取失败，将会进入`_EntryList`集合等待，此时线程状态是`BLOCKING`状态

#### `synchronized` 与对象头
在JVM中，对象在堆内存中的布局分为三块：对象头、实例数据、对齐填充。其中，对象头存储着与该对象相关的元数据（如指向方法区类`class`对象的指针）、`HashCode`、`GC`分代年龄、以及当前对象的锁定状态等；实例数据则是存储着对象真正的数据，包括父类继承过来的字段，以及自身定义的字段；对齐填充是因为JVM中要求每个对象的内存空间大小必须为8字节的整数倍，如果对象头和实例数据加一起已经是8字节的整数倍，则不存在对齐填充来补全。其中，对象头内部还分为两部分数据：第一部分用于存储对象的`HashCode`、`GC`分代年龄、锁状态等，被称为`Mark Word`，第二部分用于存储指向方法区类`class`对象的指针。（如果当前对象是数组对象的话，还会有第三部分用于存储数组长度），具体的对象内部布局结构如下图所示：
![线程状态转换](images/对象JVM布局.jpg)