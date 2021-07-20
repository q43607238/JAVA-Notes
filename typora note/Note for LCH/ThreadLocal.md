[TOC]

# ThreadLocal

##  简介

`ThreadLocal`是一个将在多线程中为每一个线程创建单独的变量副本的类; 当使用`ThreadLocal`来维护变量时, `ThreadLocal`会为每个线程创建单独的变量副本, 避免因多线程操作共享变量而导致的数据不一致的情况。如果我们希望通过某个类将状态(例如用户ID、事务ID)与线程关联起来，那么通常在这个类中定义`private static`类型的`ThreadLocal` 实例。

---

##  学习参考资料

https://www.pdai.tech/md/java/thread/java-thread-x-threadlocal.html

https://segmentfault.com/a/1190000022663697

---

##  使用场景

如下数据库管理类在单线程使用是没有任何问题的：

```java
class ConnectionManager {
    private static Connection connect = null;

    public static Connection openConnection() {
        if (connect == null) {
            connect = DriverManager.getConnection(); //获取一个数据库的连接对象
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

        // 使用connection进行sql操作

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

---

## 线程隔离原理

<img src="https://raw.githubusercontent.com/q43607238/JAVA-Notes/master/typora%20pic/ThreadLocal/ThreadLocal.png" alt="ThreadLocal结构" style="zoom:60%;" />

在`ThreadLocal`中，存在一个`ThreadLocalMap`的内部类，在`ThreadLocalMap`类中还有一个`Entry`实体类：

### Entry

值得注意的是，`ThreadLocalMap`中的`Entry`是一个弱引用，`WeakReference`引用的对象，在GC的时候**无论是否内存空间足够都会被回收。**

这里就需要提到**内存泄露**的概念，内存泄漏往往发生在**一个短生命周期的对象被一个长生命周期对象长期持有引用，将会导致该短生命周期对象使用完之后得不到释放，从而导致内存泄漏。** 例如：在`ThreadLocal`中，其本身就是一个短期生命周期对象，例如在数据库连接过程中得`connection`数据库连接对象，在执行完sql之后可能就会关闭。

1. **为什么ThreadLocalMap使用弱引用存储ThreadLocal？**

   假如使用强引用，当`ThreadLocal`不再使用需要回收时，发现某个线程中`ThreadLocalMap`存在该`ThreadLocal`的强引用，无法回收，造成内存泄漏。

   因此，使用弱引用可以防止长期存在的线程（通常使用了线程池）导致`ThreadLocal`无法回收造成内存泄漏。

2. **那通常说的ThreadLocal内存泄漏是如何引起的呢？**

   我们注意到`Entry`对象中，虽然Key(ThreadLocal)是通过弱引用引入的，但是value即变量值本身是通过强引用引入。

   这就导致，假如不作任何处理，由于`ThreadLocalMap`和线程的生命周期是一致的，当线程资源长期不释放，即使`ThreadLocal`本身由于弱引用机制已经回收掉了，但value还是驻留在线程的`ThreadLocalMap`的`Entry`中。即存在key为null，但value却有值的无效`Entry`。导致内存泄漏。

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

###  get()

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
    int i = key.threadLocalHashCode & (table.length - 1); //通过hash值计算对应key在TheadLocalMap中的table里面的下标
    Entry e = table[i]; // table是ThreadLocalMap的私有变量，是一个entry数组
    if (e != null && e.get() == key) //如果在计算出的hash坐标下直接找到了对应的key，就直接把value返回
        return e;
    else
        return getEntryAfterMiss(key, i, e); //如果在当前hash坐标下没有找到key，就调用getEntryAfterMiss
}
```

### getEntryAfterMiss()

这个方法在我们没有在预计的hash下标下找到我们的key的时候启动，往后寻找我们需要的key并返回value，同时，如果发现了key为null的值，调用`expungeStaleEntry()`来清理key为`null`的Entry。

```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table; //注意，这里由于在ThreadLocalMap中定义的Entry是一个弱引用，所以对tab做更改的时候，也会反应到table变量上去
    int len = tab.length;

    while (e != null) {
      ThreadLocal<?> k = e.get(); //这里实体e的get方法是继承自其父类的Reference的方法，返回了弱引用的对象
      if (k == key)
        return e;
      if (k == null)
        // 如果发现了key为null的值，则调用expungeStaleEntry
        expungeStaleEntry(i);
      else
        i = nextIndex(i, len);
      e = tab[i];
    }
    return null;
}
```

### expungeStaleEntry()

删除过时的实体！这个方法在从`ThreadLocalMap`的`table`里面拿`Entry`的时候发现key是`null`的时候会调用，其不仅会删除当前的无效`Entry`，也会往后遍历，删除别的key为`null`而value还未被回收的`Entry`。

```java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // 第一步，将当前key为null的坐标下的实体内容全部置为null
    tab[staleSlot].value = null; //这里把value置为null因为key为弱引用，因此被清除，但是value无法被回收，因此首先要把value置为null
    tab[staleSlot] = null; //这里是吧ThreadLocalMap里面的table对应的位置置为了null，表示这个坐标下可以存一个新种类的ThreadLocal的实体
    size--;

    // Rehash until we encounter null
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len); //从这个为空的下标staleSlot去往后遍历
         (e = tab[i]) != null; // 直到实体为null终止，即i已经到了table的边界
         i = nextIndex(i, len)) {
      ThreadLocal<?> k = e.get();
      if (k == null) {
        //如果发现key是null，则清除value，把table对应坐标置为null
        e.value = null;
        tab[i] = null;
        size--;
      } else {
        //如果key不是null的话，对当前实体进行rehash
        int h = k.threadLocalHashCode & (len - 1);
        if (h != i) {
          //如果当前坐标和我们预计的hash坐标不一样，把当前坐标置为空，把这个Entry移动到我们希望其在的h下标处
          tab[i] = null;
          while (tab[h] != null)
            //这个循环是因为，有可能我们的目标位置h已经被别的Entry占用了，在ThreadLocalMap的table中，解决hash冲突的方式就是继续往后存储，而不是像map一样可以以链表的形式累积下去
            h = nextIndex(h, len);
          tab[h] = e;
        }
      }
    }
    return i; //返回table的边界的i的值
}
```

初始化的代码`setInitialValue`为：

### setInitialValue()

```java
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
      //当发现线程内还不存在ThreadLocalMap (在Thread线程内部的变量定义名字为threadLocals)的时候，就创建一个新的Map
        createMap(t, value);
    return value;
}
//其中的getMap
ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }

//其中createMap
void createMap(Thread t, T firstValue) {
  	t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

### set()

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
            replaceStaleEntry(key, value, i); //这是一个过期的实体，调用replaceStaleEntry替换实体，并进行一些key为null的清理
            return;
        }
    }
    //如果遍历到当前的i处的实体e是一个null，则表示在这个位置table没有存东西，则直接在这个下标处存进去并且吧size++
    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold) //如果在清除过后发现，Entry数量超过了阈值，于是进行rehash
        rehash();
}
```

### cleanSomeSlots()

在这个方法中，

```java
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
      i = nextIndex(i, len);
      Entry e = tab[i];
      if (e != null && e.get() == null) {
        n = len;
        removed = true;
        i = expungeStaleEntry(i);
      }
    } while ( (n >>>= 1) != 0);
    return removed;
}
```

### replaceStaleEntry（）

替换过期的实体；在set方法中调用。

```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value, int staleSlot) { //传入的staleSlot就是在set方法中发现的key为null的下标
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    //往前寻找，查找最前的一个无效key
    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len)) //这个循环往前寻找到一个tab不为null而key为null的下标i
        if (e.get() == null) 
            slotToExpunge = i;
  
	//从set方法发现的key为null的位置的下一个，继续往后遍历table
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            //往后遍历的过程中，如果发现了对应的key，那就吧这个位置的entry和之前set方法中发现的key为null的entry互换位置
            e.value = value; //把当前Entry的value设置为我们要set进来的value

            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            // Start expunge at preceding stale entry if it exists
            if (slotToExpunge == staleSlot)
                slotToExpunge = i; //如果在最开始的往前遍历没有无效的key，那因为我们互换了k==key时的i和staleSlot的位置，所以现在i处的key是null，因此从i开始进行cleanSomeSlots
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return; //任务完成，直接return
        }

        //如果在往后遍历的时候发现存在key为null并且slotToExpunge往前寻找的时候没找到，所以，不断更换slotToExpunge的值
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    // 如果在往后遍历中没有找到我们需要set的目标key，则在set方法发现的key为null的地方，创建新的Entry并赋值
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    // If there are any other stale entries in run, expunge them
    if (slotToExpunge != staleSlot) //这里是因为如果在往前遍历的时候有key为null的地方，就会导致slotToExpunge != staleSlot，并引发一次清理；
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```









