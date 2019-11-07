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


#### `synchronized` 与对象头