## Java多线程之`ThreadLocal`

#### 前言

在多线程环境下进行开发时，经常会遇到这样子的一种场景，就是在使用某个非线程安全的类时，鉴于线程安全问题，因此会在每次使用时，创建一个新的实例对象，典型的例子就是`SimpleDateFormat`日期格式化类。即使是对于同一个线程而言，也是在每次使用时创建一个对象实例，这无疑会产生对象的重复创建情况，并因此带来一定的性能损耗，在`Web`应用中尤为常见。

线程安全是针对于多个线程共享使用同一实例对象而言，那么对于单个线程而言，或者更具体点来说，在单个线程内部，是不会存在线程安全问题的。那么能不能存在这样的一种机制：每个线程独享一个专属的实例对象，线程间不再共享实例对象。言外之意就是为每个线程创建一个单独的对象。`ThreadLocal`就是这样子的一种机制，为了每个线程创建一个单独的专属实例对象的机制。

#### ThreadLocal 与 ThreadLocalMap

在<<阿里巴巴Java开发规范>>中的并发处理部分中，有一个很好的关于`ThreadLocal`的示例，这里就直接搬运过来，如下所示：

```java
private static final ThreadLocal<DateFormat> df = new ThreadLocal<DateFormat>() {
    @Override
    protected DateFormat initialValue() {
        return new SimpleDateFormat("yyyy-MM-dd");
    }
};
```
在上述代码中，能看到通过`ThreadLocal`对`SimpleDateFormat`进行了封装，这样子就能够保证在多线程环境下，每个线程独享一个专属的`SimpleDateFormat`实例对象，不会产生线程安全问题。并且对于同一个线程内部而言，在每次使用时，都能够复用之前创建的`SimpleDateFormat`实例对象。

那么会有个疑问，实例对象是怎么和线程关联起来的呢，通过剖析`ThreadLocal`、`Thread`源码，可以看出，在每个线程内部都有这么两个属性：`threadLocals`、`inheritableThreadLocals`，具体源码（`java.lang.Thread`）如下：
```java
/*
 * ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class. 
 */
ThreadLocal.ThreadLocalMap threadLocals = null;

/*
 * InheritableThreadLocal values pertaining to this thread. This map is
 * maintained by the InheritableThreadLocal class.
 */
ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```
*为了方便剖析`ThreadLocal`的底层实现，这里先忽略`inheritableThreadLocals`属性，会在后面会专门提到它的作用，下面的内容都是围绕着`threadLocals`属性展开的。*

使用实例对象时，通过 `ThreadLocal#get()` API进行获取，具体源码（`java.lang.ThreadLocal`）如下所示
```java
public T get() {

    //获取当前线程
    Thread t = Thread.currentThread();

    //从线程中获取threadLocals属性指向的ThreadLocalMap对象结构
    ThreadLocalMap map = getMap(t);
    if (map != null) {

        //获取对应的实例对象
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }

    //获取不到实例对象，设置默认值并返回
    return setInitialValue();
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

源码中可以看出：每次获取实例对象都是从`threadLocals`属性指向的`ThreadLocalMap`对象结构中获取。其实，`ThreadLocalMap`就是一个底层采用数组存储的`Map`结构，但是没有直接继承`Map`接口，数组的每个元素被包装成`ThreadLocalMap.Entry`对象。既然是从`Map`结构中获取，那肯定是根据一个`Key`来获取，这个`Key`就是`ThreadLocal`本身，也许很好奇，`ThreadLocal`对象怎么会是一个`Key`呢，没错它就是一个`Key`，可以唯一标记一个实例对象，用于从线程对象中的`threadLocals`属性中获取对应的实例对象。

每次个`ThreadLocal`对象被创建时，都会为创建的`ThreadLocal`对象生成一个`threadLocalHashCode`作为其唯一的标识Code，并且每次生成的`ThreadLocal`对象的这个`threadLocalHashCode`唯一且保持固定步长递增，具体源码如下：
```java
private final int threadLocalHashCode = nextHashCode();

/**
  * The next hash code to be given out. Updated atomically. Starts at
  * zero.
  */
private static AtomicInteger nextHashCode =
    new AtomicInteger();

/**
  * The difference between successively generated hash codes - turns
  * implicit sequential thread-local IDs into near-optimally spread
  * multiplicative hash values for power-of-two-sized tables.
  */
private static final int HASH_INCREMENT = 0x61c88647;

/**
  * Returns the next hash code.
  */
private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```
因此，可以将`ThreadLocal`简单的理解为一个具有唯一标识Code的`Key`，说完了`ThreadLocal`，接着说`ThreadLocalMap`，就如同上面提到的一样，`ThreadLocalMap`是一个底层采用数组结构存储的`Map`结构，查看`ThreadLocalMap`相关源码如下：
```java
static class ThreadLocalMap {

    static class Entry extends WeakReference<ThreadLocal<?>> {

        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }

    private static final int INITIAL_CAPACITY = 16;

    private Entry[] table;

    private int size = 0;

    private int threshold;

    .....
}
```
为了方便查看，将源码中无关的代码以及"绿化区"移除掉了，通过上述源码可以看出，`ThreadLocalMap`的初始容量是16，元素存储在`Entry`类型的一个数组`table`中，这里的`Entry`不仅仅是一个键值对，还是一个弱引用类型，并且弱引用的是`Key`，即表明是一个弱键型的类，这里的`Key`也就是上面提到的`ThreadLocal`。我们知道弱引用不同于强引用，弱引用指向的对象会在JVM发生GC时，被回收掉，因此`ThreadLocal`还有一个好处就是我们不用担心创建的实例对象的内存回收问题。

#### 实例对象的存取

接下来，结合`ThreadLocal`、`ThreadLocalMap`来一起分析下，如何存取`ThreadLocal`保存的实例对象，先看取的过程，具体源码(`java.lang.ThreadLocal.ThreadLocalMap#getEntry()`)如下:
```java
private Entry getEntry(ThreadLocal<?> key) {
    //根据 key 的 hashCode 计算对应的下标位置
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];

    //判断是否存在哈希碰撞
    if (e != null && e.get() == key)
        return e;
    else
        //如果存在哈希碰撞，则采用线性探测法往后查询
        return getEntryAfterMiss(key, i, e);
}

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}


```
通过源码可以看到，在取对应的实例对象时，先通过`Key`的`threadLocalHashCode`计算出对应的下标位置，然后寻找对应下标位置的元素，最后对比取出元素的`Key`是否和目标`Key`是同一个，如果不是，则表明发生了哈希碰撞，采用线性探测法向后逐个查找目标`Key`。这里为什么说是线性探测法呢，是因为在发现对应位置的元素不是目标`Key`时，则会通过`nextIndex(int i, int len)`计算出下一个下标位置，`nextIndex(int i, int len)`的源码(`java.lang.ThreadLocal.ThreadLocalMap#nextIndex()`)如下，其实就是当前下标位置往后移一位，也就是线性探测法。此外，我们在上述代码中还看到，当目标位置元素的`Key`为null时，会将该位置上的元素从`ThreadLocalMap`中驱逐掉，关于具体的驱逐过程会在存放过程后面阐述，这里只需要知道取实例对象时，会产生过期元素驱逐即可。
```java
/**
  * Increment i modulo len.
  */
private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}
````
关于如何取实例对象的过程也有所了解了，接下来阐述一下如何存放实例对象，存放和读取的过程是类似的，具体的存放源码如下所示：
```java
private void set(ThreadLocal<?> key, Object value) {

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
            e != null;
            e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```
由源码可以看出存放时，同样是先根据`Key`的`threadLocalHashCode`计算出对应的下标位置，取出对应位置元素比较，如果不存在哈希碰撞，则存入元素返回，如果存在哈希碰撞，则采用线性探测法，继续向后寻找空位存放。此外，在每次存放完成后，会尝试从`ThreadLocalMap`中清除掉部分过期的实例对象，如果没有过期的实例对象，并且当前线程持有的实例对象数量超过阈值时，则进行`rehash()`计算。关于`Key`驱逐、以及`rehash()`过程会在下面具体阐述，这里只需要知道存放实例对象时，也会产生驱逐过期元素即可。

#### 关于过期实例对象的驱逐
上面提到ThreadLocalMap.Entry类是一个弱键类，并且弱键的是`Key`，意味着在发生GC时，会导致部分无外界强引用的`Key`被回收掉，一旦`Key`被回收掉，也就意味着这个`Key`不复存在，不能再获取到这个`Entry`保存的实例对象，也就说这个实例对象过期了，需要从`ThreadLocalMap`中被剔除掉，在上面的存取过程中都有相应的驱逐操作，具体的驱逐源码（`java.lang.ThreadLocal.ThreadLocalMap#expungeStaleEntry(int staleSlot)`）如下：
```java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // expunge entry at staleSlot
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // Rehash until we encounter null
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
            (e = tab[i]) != null;
            i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;

                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
````
代码很长有些难懂，简单来说，这个操作就是对单个位置上的过期元素进行驱逐，以及对排在其后面的元素进重新计算下标位置的过程，至于为什么需要对排在其后面的元素重新计算下标位置，是因为存放时如果发生了哈希碰撞，会按照线性探测法往后排放，现在前面位置的元素被驱逐掉了，后面的元素理应重新计算位置，往前排。在深入一点思考，为什么不是直接把后面的元素往前移动一位，而是重新计算下标位置呢，这一点是因为后面的元素不一定是和因为和过期元素发生哈希碰撞导致的往后排，也有可能是和排在过期元素后面的其他元素发生哈希碰撞而导致的后排，所以采用重新计算下标位置是最正确的处理方式。

除了对特定位置的元素进行驱逐，还有全局驱逐，源码（`java.lang.ThreadLocal.ThreadLocalMap#expungeStaleEntries()`）如下，其实就是对每个位置上的元素进行逐个驱逐，就不在具体分析了
```java
private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
        if (e != null && e.get() == null)
            expungeStaleEntry(j);
    }
}
````

对于`rehash()`操作，具体的源码如下所示：
```java
private void rehash() {
    expungeStaleEntries();

    if (size >= threshold - threshold / 4)
        resize();
}

private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;

    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                e.value = null; // Help the GC
            } else {
                int h = k.threadLocalHashCode & (newLen - 1);
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e;
                count++;
            }
        }
    }

    setThreshold(newLen);
    size = count;
    table = newTab;
}
```
