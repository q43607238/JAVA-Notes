[TOC]

# ThreadLocal

## 简介

`ThreadLocal`是一个将在多线程中为每一个线程创建单独的变量副本的类; 当使用`ThreadLocal`来维护变量时, `ThreadLocal`会为每个线程创建单独的变量副本, 避免因多线程操作共享变量而导致的数据不一致的情况。如果我们希望通过某个类将状态(例如用户ID、事务ID)与线程关联起来，那么通常在这个类中定义`private static`类型的`ThreadLocal` 实例。

## 学习参考资料

https://www.pdai.tech/md/java/thread/java-thread-x-threadlocal.html

https://segmentfault.com/a/1190000022663697

## 使用场景

如下数据库管理类在单线程使用是没有任何问题的：

```java
class ConnectionManager {
    private static Connection connect = null;

    public static Connection openConnection() {
        if (connect == null) {
            connect = DriverManager.getConnection();
        }
        return connect;
    }

    public static void closeConnection() {
        if (connect != null)
            connect.close();
    }
}
```

很显然，在多线程中使用会存在线程安全问题：

1. 这里面的2个方法都没有进行同步，很可能在`openConnection`方法中会多次创建`connect`；
2. 由于`connect`是共享变量，那么必然在调用`connect`的地方需要使用到同步来保障线程安全，因为很可能一个线程在使用`connect`进行数据库操作，而另外一个线程调用`closeConnection`关闭链接。

假如每个线程中都有一个`connect`变量，各个线程之间对`connect`变量的访问实际上是没有依赖关系的，即一个线程不需要关心其他线程是否对这个`connect`进行了修改的。即我们可以在每一次需要建立`connection`的时候都新建一个`connection`对象：

```java
class Dao {
    public void insert() {
        ConnectionManager connectionManager = new ConnectionManager(); //每一次都新建一个connection
        Connection connection = connectionManager.openConnection();

        // 使用connection进行操作

        connectionManager.closeConnection();
    }
}
```

这样处理确实也没有任何问题，由于每次都是在方法内部创建的连接，那么线程之间自然不存在线程安全问题。但是这样会有一个致命的影响：导致服务器压力非常大，并且严重影响程序执行性能。由于在方法中需要频繁地开启和关闭数据库连接，<span style='color:red'>**这样不仅严重影响程序执行效率，还可能导致服务器压力巨大。**</span>



**那么这种情况下使用ThreadLocal是再适合不过的了**，因为ThreadLocal在每个线程中对该变量会创建一个副本，即每个线程内部都会有一个该变量，且在线程内部任何地方都可以使用，线程之间互不影响，这样一来就不存在线程安全问题，也不会严重影响程序执行性能:

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

public class ConnectionManager {
    private static final ThreadLocal<Connection> dbConnectionLocal = new ThreadLocal<Connection>() { //如果我们希望通过某个类将状态(例如用户ID、事务ID)与线程关联起来，那么通常在这个类中定义private final static类型的ThreadLocal 实例。
        @Override
        protected Connection initialValue() {
            try {
                return DriverManager.getConnection("", "", "");
            } catch (SQLException e) {
                e.printStackTrace();
            }
            return null;
        }
    };

    public Connection getConnection() {
        return dbConnectionLocal.get(); //从当前的ThreadLocal获取对应线程的connection副本，在get中会涉及到获取，或者线程是第一次建立连接的话，就会调用setInitialValue()新建一个connection并存储在ThreadLocal里面
    }
}
```

## 线程隔离原理

<img src="https://raw.githubusercontent.com/q43607238/JAVA-Notes/master/typora%20pic/ThreadLocal/ThreadLocal.png" alt="ThreadLocal结构" style="zoom:30%;" />

在`ThreadLocal`中，存在一个`ThreadLocalMap`的内部类，在`ThreadLocalMap`类中还有一个`Entry`实体类：

### Entry

值得注意的是，`ThreadLocalMap`中的`Entry`是一个弱引用，`WeakReference`引用的对象，在GC的时候**无论是否内存空间足够都会被回收。**

这里就需要提到**内存泄露**的概念，内存泄漏往往发生在**一个短生命周期的对象被一个长生命周期对象长期持有引用，将会导致该短生命周期对象使用完之后得不到释放，从而导致内存泄漏。** 例如：在`ThreadLocal`中，其本身就是一个短期生命周期对象，例如在数据库连接过程中得`connection`数据库连接对象，在执行完sql之后可能就会关闭。

1. **为什么ThreadLocalMap使用弱引用存储ThreadLocal？**

   假如使用强引用，当ThreadLocal不再使用需要回收时，发现某个线程中ThreadLocalMap存在该ThreadLocal的强引用，无法回收，造成内存泄漏。

   因此，使用弱引用可以防止长期存在的线程（通常使用了线程池）导致ThreadLocal无法回收造成内存泄漏。

2. **那通常说的ThreadLocal内存泄漏是如何引起的呢？**

   我们注意到Entry对象中，虽然Key(ThreadLocal)是通过弱引用引入的，但是value即变量值本身是通过强引用引入。

   这就导致，假如不作任何处理，由于ThreadLocalMap和线程的生命周期是一致的，当线程资源长期不释放，即使ThreadLocal本身由于弱引用机制已经回收掉了，但value还是驻留在线程的ThreadLocalMap的Entry中。即存在key为null，但value却有值的无效Entry。导致内存泄漏。

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k); //这里表示对ThreadLocal的弱引用
        value = v;
    }
}
```

先看`ThreadLocal`的`get`方法：

### get

```java
public T get() { //T在本例中就是connection，返回值，调用这个方法的是dbConnectionLocal，即ThreadLocal
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t); //获取对应t线程内的ThreadLoclaMap
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this); //这个this指的是dbConnectionLocal，这里是因为一个线程可能有多个对象的副本，在这次调用，需要找到的connection的副本，所以要根据this来选定对应的entry
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value; //在这个entry中，value就是我们需要的connection
            return result;
        }
    }
    return setInitialValue();
}

//getEntry是ThreadLocalMap的方法
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i]; // table是ThreadLocalMap的私有变量，是一个entry数组
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
```

初始化的代码`setInitialValue`为：

### setInitialValue

```java
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
```

### set

```java
private void set(ThreadLocal<?> key, Object value) {

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1); //从默认的hash计算下标的地方出发，往后遍历

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i); //这是一个过期的实体，调用replaceStaleEntry替换实体
            return;
        }
    }
    //如果遍历到当前的i是一个null，则直接在这个下标处存进去并且吧size++
    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

### replaceStaleEntry（）

替换过期的实体；

```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value, int staleSlot) { //传入的staleSlot就是在set方法中发现的key为null的下标
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    //往前寻找，查找最前的一个无效key
    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))
        if (e.get() == null)
            slotToExpunge = i;
	//从set方法发现的key为null的位置的下一个，继续往后遍历table
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            //往后遍历的过程中，如果发现了对应的key，那就吧这个位置的entry和之前key为null的entry互换位置
            e.value = value;

            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            // Start expunge at preceding stale entry if it exists
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        // If we didn't find stale entry on backward scan, the
        // first stale entry seen while scanning for key is the
        // first still present in the run.
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    // If key not found, put new entry in stale slot
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    // If there are any other stale entries in run, expunge them
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

## 未完待续，一些回收方法还没看懂







