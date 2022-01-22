---
title: 'Understand ThreadLocal'
date: 2021-8-9 00:47:01 +0800
author: '5akura'
categories:
    - Java_Base
---

[TOC]

## 三、简单捋捋 ThreadLocal 这玩意

看了好些书，每次都能看到这个东西（`ThreadLocal`）,每次都是弄清楚个大概，然后过几天再遇到就忘得干干净净，今天不妨就坐下来好好捋捋这个东西，并记录下来！

由于笔者对并发编程接触还停留在基础或者说理论阶段，大多参考学习手边的书籍，所以笔记中可能存在问题，望理解~ 参考书籍：《码出高效 Java 开发手册》、《深入理解 Java 虚拟机》、《并发编程的艺术》

---

### 这玩意是什么？能干什么？

> `ThreadLocal`是为了**解决并发环境下数据共享问题**而设计的类。

了解并发的各位都知道，一份数据如果同时被多个线程同时访问/修改，难免会出现一些问题！针对各种问题出现了很多种解决方式：

1. 可见性问题：保证共享数据的可见性，使用`volatile`修饰变量 or 加锁
2. 数据共享问题： 线程封闭，不共享数据
3. 并发修改问题：使用不可变对象

而我们今天要聊的`ThreadLocal`就是解决方式会利用的类！我们来看看官方文档上对其的描述：

> _This class provides thread-local variables. These variables differ from their normal counterparts in that each thread that accesses one (via its `get` or `set` method) has its own, independently initialized copy of the variable. `ThreadLocal` instances are typically private static fields in classes that wish to associate state with a thread (e.g., a user ID or Transaction ID)._

简单说就是：使用这个类会为每个线程创建一个局部变量，不同于普通变量，这些变量通过在各个线程中通过调用自己 ThreadLocal 对象的`get`、`set`方法来获取或者设置变量值，而不会影响其他线程中的数据值。通常被用于保存类中私有的、静态的那些希望将状态与线程关联起来的字段！

<img src="https://picbed-sakura.oss-cn-shanghai.aliyuncs.com/notePic/image-20210809171025760.png" alt="image-20210809171025760" style="zoom:80%;" />

因为线程之间是不会直接共享数据的，所以每个线程单独维护一份数据是不会出现多线程共享数据的问题的。这都很好理解，关键是我们要摸清楚它是如何做到这一点的！所以有以下几个问题：

-   如何做到每个线程单独一份数据的？
-   单个线程访问自己那份数据的过程是怎样的？

---

### 学会使用 ThreadLocal，先来过把瘾

因为 ThreadLocal 的本心就是不希望出现线程共享数据。

我们用一个多人游戏（大富翁）的案例进行测试。每名玩家视为一个线程，玩家的起始货币数量都是相同在游戏中各个玩家货币独立计算，并且拥有的“房产”也是相互独立的，所以我们需要避免这两个数据在多线程间共享，并且需要将数据值（状态）与线程关联起来。所以就需要使用`ThreadLocal`。

在使用之前，建议先去看看[【官方文档及其案例】](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/ThreadLocal.html)：这里我简单截个图
<img src="https://picbed-sakura.oss-cn-shanghai.aliyuncs.com/notePic/image-20210809182213599.png" alt="image-20210809182213599" style="zoom:80%;" />

#### step1: 创建

如果你希望在 ThreadLocal 的初始化时做一些额外操作（例如：赋初始值）你可以使用 ThreadLocal 的静态方法`withInitial`，传入一个“生产者”函数从而得到 ThreadLocal 对象：

```java
private static final Integer INIT_MONEY = 50_000;

private static final ThreadLocal<Integer> INIT_MONEY_LOCAL = ThreadLocal.withInitial(() -> INIT_MONEY);
```

或者说如果你觉得这样写不便于理解，可以直接使用`new ThreadLocal<>(){ ... }`然后按需要覆写`initialValue()`方法来自定义初始值设置和其他操作！

```java
private static final Integer INIT_HOUSES = 1;

private static final ThreadLocal<Integer> INIT_HOUSES_LOCAL = new ThreadLocal<>() {
    @Override
    protected Integer initialValue() {
        // 自定义初始化操作
        return super.initialValue();
    }

    @Override
    public Integer get() {
        return super.get();
    }

    @Override
    public void set(Integer value) {
        super.set(value);
    }

    @Override
    public void remove() {
        super.remove();
    }
};
```

==但是要注意的是：两种方式实现的自定义初始化操作即`initialValue()`方法，都只会在第一次`get()`方法被调用之前执行！但是如果 get 前调用了 set，则不会执行！==
（两种创建 ThreadLocal 对象的方式殊途同归，建议看看源码：）

所以说如果你没有覆写`initialValue()`也不是使用`withInitial()`创建 ThreadLocal 时，就会执行默认的`initialValue()`:

```java
protected T initialValue() {
    return null;
}
```

那这样，你首次 get(没有调用 set 进行设值)到的值就只能是 null！

#### step2：设值、取值

这就不用我说了吧，`get()`、`set()`。
但是千万不要小瞧了这两个方法，它们就是我们谜题的入口！我先把源码贴出来：

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}

public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        map.set(this, value);
    } else {
        createMap(t, value);
    }
}
```

#### step3: 销毁

为当前线程删除此线程局部变量的值，销毁当下一次再次访问`get()`此局部变量时会重新执行初始化操作即`initialValue()`，除非你在 get 前已经调用过 set 了。

```java
private static final Integer INIT_HOUSES = 1;

private static final ThreadLocal<Integer> INIT_HOUSES_LOCAL = new ThreadLocal<>() {
    @Override
    protected Integer initialValue() {
        System.out.println("Initializing....");
        return INIT_HOUSES;
    }
};

public static void main(String[] args) {

    System.out.println(INIT_HOUSES_LOCAL.get());
    INIT_HOUSES_LOCAL.set(20)
    INIT_HOUSES_LOCAL.remove();
    System.out.println(INIT_HOUSES_LOCAL.get());

}


/*
output:

    Initializing....
    1
    Initializing....
    1
*/
```

#### 完整 Demo

```java
public class RichManGameByThreadLocal {
    private static final Integer INIT_MONEY = 50_000;
    private static final Integer INIT_HOUSES = 1;

    private static final ThreadLocal<Integer> INIT_MONEY_LOCAL = ThreadLocal.withInitial(() -> INIT_MONEY);
    private static final ThreadLocal<Integer> INIT_HOUSES_LOCAL = ThreadLocal.withInitial(() -> INIT_HOUSES);

    public static void main(String[] args) {
        Player lee = new Player("Lee") {
            @Override
            public void run() {
                // Lee用20,000购买了一套房产
                INIT_MONEY_LOCAL.set(INIT_MONEY_LOCAL.get() - 20_000);
                INIT_HOUSES_LOCAL.set(INIT_HOUSES_LOCAL.get() + 1);

                System.out.println(
                        Thread.currentThread().getName()
                                + ": I have $ " + INIT_MONEY_LOCAL.get()
                                + ", and " + INIT_HOUSES_LOCAL.get() + " house(s)!");
            }
        };

        Player mary = new Player("Mary") {
            @Override
            public void run() {
                // Mary用10,000购买了一套房产, 然后卖出了一套房产获利15,000
                INIT_MONEY_LOCAL.set(INIT_MONEY_LOCAL.get() - 10_000);
                INIT_HOUSES_LOCAL.set(INIT_HOUSES_LOCAL.get() + 1);

                INIT_HOUSES_LOCAL.set(INIT_HOUSES_LOCAL.get() - 1);
                INIT_MONEY_LOCAL.set(INIT_MONEY_LOCAL.get() + 15_000);

                System.out.println(
                        Thread.currentThread().getName()
                                + ": I have $ " + INIT_MONEY_LOCAL.get()
                                + ", and " + INIT_HOUSES_LOCAL.get() + " house(s)!");
            }
        };

        Player bob = new Player("Bob") {
            @Override
            public void run() {
                // Bob出售了一套房产，获利20_000
                INIT_HOUSES_LOCAL.set(INIT_HOUSES_LOCAL.get() - 1);
                INIT_MONEY_LOCAL.set(INIT_MONEY_LOCAL.get() + 20_000);

                System.out.println(
                        Thread.currentThread().getName()
                                + ": I have $ " + INIT_MONEY_LOCAL.get()
                                + ", and " + INIT_HOUSES_LOCAL.get() + " house(s)!");
            }
        };

        lee.start();
        mary.start();
        bob.start();

        try {
            TimeUnit.SECONDS.sleep(3);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    private static class Player extends Thread {
        public Player(String name) {
            super(name);
        }
    }

}

/*
output:

Bob: I have $ 70000, and 0 house(s)!
Lee: I have $ 30000, and 2 house(s)!
Mary: I have $ 55000, and 1 house(s)!
*/
```

这是一个非常非常简陋的 Demo，游戏流程用剧本的方式呈现出来。但是可以看到 ThreadLocal 确实起到了避免线程之间共享数据的效果！下一步我们就要来看看它是如何实现的了！

---

### 看看 ThreadLocal 核心源码

首先我觉得有必要先了解以下其类结构，因为`ThreadLocal`还藏着一些内部类：

<img src="https://picbed-sakura.oss-cn-shanghai.aliyuncs.com/notePic/image-20210809203746549.png" alt="image-20210809203746549" style="zoom:80%;" />

我们此前已经暴露了很多问题，下面逐个说明

#### 两种创建方式，殊途同归？

使用`new ThreadLocal() { ... }`创建的方式，这里就省略了。
我们只看`ThreadLocal.withInitial(() -> { ... })`方式创建的过程：

`withInitial()`的源码：

```java
public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
    return new SuppliedThreadLocal<>(supplier);
}
```

现在我们将目光移到`SuppliedThreadLocal`这个类身上，它是集成自 ThreadLocal 的，并且是 ThreadLocal 的一个静态内部类：

```java
static final class SuppliedThreadLocal<T> extends ThreadLocal<T> {

    private final Supplier<? extends T> supplier;

    SuppliedThreadLocal(Supplier<? extends T> supplier) {
        this.supplier = Objects.requireNonNull(supplier);
    }

    @Override
    protected T initialValue() {
        return supplier.get();
    }
}
```

> 这里不熟悉函数式接口的小伙伴可能会对这个`supplier.get()`感到疑惑了， 但这不是重点，你可以将其理解为是生产者的生产方法，我们在使用`withInitial()`时，传入的参数函数就是对这个生产方法的实现。所以**调用`supplier.get()`就相当于是调用我们传入的参数函数`() -> { ... }`**

这样看来，确实最终都是覆写了`ThreadLocal`的`initialValue()`方法！

#### 先调用了 set(), 就不会触发 initialValue()？

通过官方文档可知，会不会触发`initialValue()`，还得看`get()`。我们再把 get 的源码贴一遍：

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

这段代码的大致逻辑就是，先查 map，如果 map 中有值（e != null）就返回其值。否则就调用`setInitialValue()`。

我们就先来看看`setInitialValue()`方法：

```java
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        map.set(this, value);
    } else {
        createMap(t, value);
    }
    if (this instanceof TerminatingThreadLocal) {
        TerminatingThreadLocal.register((TerminatingThreadLocal<?>) this);
    }
    return value;
}
```

第一行就调用了`initialValue()`，然后将获得的初始值设置在一个 map 中，最后返回这个初始值。

然后我再来看看 set 的代码：

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        map.set(this, value);
    } else {
        createMap(t, value);
    }
}
```

通过查看`get`、`setInitialValue()`、`set()`三个方法的代码，不难看出引发这个情况的主要还是因为取值、设值前都会先检查 map 中是否已经有值。这也是为了防止反复执行初始化操作！

---

### 新朋友, ThreadLocalMap

在核心源码的阅读过程中，你一定发现了这个新的类：`ThreadLocalMap`，并且在类结构图中，看得出它是 ThreadLocal 的一个内部类。

#### 初次见面，揭开面纱

而它的实例对象在 ThreadLocal 中通过`getMap()`并传入一个`Thread`对象作为参数获取。

为什么要传一个 Thread 对象呢？并且这个 Thread 还是当前执行线程的 Thread 对象，难道这里就是 ThreadLocal 实现数据线程独立的“要害”吗？！来看看`getMap()`的代码：

```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

WTF?!

吓得我直接打开了`Thread`的源码，并且找到了这个属性：

```java
/* ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;
```

注释说：这个属性表示着与此线程关联的 ThreadLocal 值，这个映射表由 ThreadLocal 进行管理。

话不多说，对我们的 Demo 进行调试，来看看 Thread 对象中这个属性中到底装的什么东西：

<img src="https://picbed-sakura.oss-cn-shanghai.aliyuncs.com/notePic/ThreadLocalMap-Debug.gif" alt="ThreadLocalMap-Debug" style="zoom:80%;" />

这个`ThreadLocalMap`对象中，有三个属性：`table`、`size`、`threshold`。这结构妥妥的一个容器嘛，而且很像 HashMap，有兴趣的可以去对比一下。

除此以外你还会发现，`table`是一个`Entry[]`，查看类结构图发现这个`Entry`是 ThreadLocalMap 的内部类，是其定义的存放数据的类型（格式）。【就很像 HashMap 中`table`是一个`Node[]`，而`Node`是 HashMap 的一个静态内部类。】

到这里我们整理一下：

1. ThreadLocalMap 是 ThreadLocal 的静态内部类，Entry 是 ThreadLocalMap 的静态内部类。
2. `ThreadLocal`对象在 get/set 时，都需要调用`getMap`取出当前线程的 Thread 对象中的`threadLocals`属性的值即一个`ThreadLocalMap`对象。
3. 每个线程中 Thread 对象维护的 ThreadLocalMap 中都存放着若干与当前线程关联的 ThreadLocal 值。并且每个值都是用`Entry`对象存储的，在 ThreadLocalMap 对象中使用数组来保管这些 Entry。

下图，参看《码出高效 Java 开发手册》p263 图 7-10

<img src="https://picbed-sakura.oss-cn-shanghai.aliyuncs.com/notePic/image-20210810134011687.png" alt="image-20210810134011687" style="zoom:80%;" />

简单总结一下就是：**每个 Thread 对象有且仅有一个 ThreadLocalMap 对象。这个 ThreadLocalMap 对象中利用 Entry[]存储了所有与当前线程相关的 ThreadLocal 值，ThreadLocal 本身不存储任何数据！**

#### 敞开心扉，展示源码

四个属性：

```java
/**
 * The initial capacity -- MUST be a power of two.
 */
private static final int INITIAL_CAPACITY = 16;

/**
 * The table, resized as necessary.
 * table.length MUST always be a power of two.
 */
private Entry[] table;

/**
 * The number of entries in the table.
 */
private int size = 0;

/**
 * The next size value at which to resize.
 */
private int threshold; // Default to 0
```

看到`threshold`就很自然想到扩容，它的扩容方式与 HashMap 很相似：

```java
private void setThreshold(int len) {
    threshold = len * 2 / 3;
}

private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;

    for (Entry e : oldTab) {
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

这部分我就不深入了，包括`rehash()`等。

我们主要任务还是看 ThreadLocalMap 的创建和取值、设值！

##### 创建 ThreadLocalMap

源码中提供了两个构造函数：

```java
/**
 * Construct a new map initially containing (firstKey, firstValue).
 * ThreadLocalMaps are constructed lazily, so we only create
 * one when we have at least one entry to put in it.
 */
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}

/**
 * Construct a new map including all Inheritable ThreadLocals
 * from given parent map. Called only by createInheritedMap.
 *
 * @param parentMap the map associated with parent thread.
 */
private ThreadLocalMap(ThreadLocalMap parentMap) {
    Entry[] parentTable = parentMap.table;
    int len = parentTable.length;
    setThreshold(len);
    table = new Entry[len];

    for (Entry e : parentTable) {
        if (e != null) {
            @SuppressWarnings("unchecked")
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
            if (key != null) {
                Object value = key.childValue(e.value);
                Entry c = new Entry(key, value);
                int h = key.threadLocalHashCode & (len - 1);
                while (table[h] != null)
                    h = nextIndex(h, len);
                table[h] = c;
                size++;
            }
        }
    }
}
```

但是到现在我们都没有提到过在哪里创建 ThreadLocalMap 对象，Thread 对象中默认是`null`。不知道细心的你有没有发现`ThreadLocal`的`set()`、`setInitialValue()`中有一个**`createMap()`**。

```java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

通过 Demo 调试，也能看到使用第一种创建方式比较频繁，我们就暂时只学习第一种创建过程：

1. 首先创建了一个初始容量大小 16 的 Entry 数组
2. 使用传入的 ThreadLocal 对象的`threadLocalHashCode`和 15（初始容量 16 - 1）做了一次与（&）运算，相当于一个简单的散列函数。然后得到散列值`i`，作为初始值的存放数组的下标。
3. 在 table 数组中下标位置`i`，创建一个 Entry 对象，传入的`firstValue`可以一直溯源到`set()`、`setInitialValue()`方法！

其实这个过程中我们忽略了看似简单其实很重要很复杂的`Entry`
涉及到**弱引用（WeakReference）**，我们后面有时间再聊

```java
/**
 * The entries in this hash map extend WeakReference, using
 * its main ref field as the key (which is always a
 * ThreadLocal object).  Note that null keys (i.e. entry.get()
 * == null) mean that the key is no longer referenced, so the
 * entry can be expunged from table.  Such entries are referred to
 * as "stale entries" in the code that follows.
 */
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

##### 取值、设值

在学习 ThreadLocal 使用过程中被我草草略过的`get()`、`set()`其实都离不开 ThreadLocalMap 中提供的取值、设值的方法，例如 get 中就调用了`map.getEntry(this)`、set 中调用了`map.set(this. value)`

取值`getEntry(ThreadLocal<?> key)`

```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
```

由于本菜鸟知识受限，我们就不过多深入分析了，大致看一下就是计算出散列值后取出 Entry 数组中对应下标的 Entry 对象直接返回，若没找到则调用其他方法尝试寻找。【至于如何保证扩容后散列值也随之规律变化，这就涉及到扩容和 rehash 的内容，我就跳过啦：) 】

设值`set(ThreadLocal<?> key, Object value)`

```java
private void set(ThreadLocal<?> key, Object value) {

    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.

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

这就更复杂了，涉及到 hash 碰撞的问题解决和 rehash 的操作。有能力或者有兴趣的小伙伴可以深挖一下，这里我就不展开了，毕竟，我很菜的 T_T

#### 江湖再见，总结

总体来说，我们如果想要取到某个线程的某一个 ThreadLocal 值，需要参与的对象有：

-   代表当前线程的 Thread 对象
-   Thread 对象中保存的 ThreadLocalMap 对象
-   代表线程本地变量的 ThreadLocal 对象

就好像一家小孩过年了找大人寻要礼物，大人们根据寻要情况分别为老大、老二、老三准备干个礼物箱，礼物箱有不同颜色（每种颜色对应一位家长的礼物，例如爸爸是蓝色，妈妈是红色），每种颜色的礼物箱每个人最多只能有一个！颜色相同的箱子密码是通用的！而这些箱子的密码只有爷爷奶奶、爸爸妈妈自己知道，孩子们要从礼物箱里面取东西或者放东西进去都只能找家长才打开箱子。

小孩就是 Thread，他们每个人自己收到的所有礼物箱构成 ThreadLocalMap，每个箱子就是一个 Entry，每位家长就是 ThreadLocal。

例如老二向爸爸许愿了礼物，爸爸把礼物箱放进了老二的房间里，老二想要拿到爸爸送的礼物就必须找爸爸，然后才能打开箱子取出礼物！

---

### 算不上总结的总结

从总体上来说，ThreadLocal 实现线程数据独立的方式很容易理解，但是本篇笔记比较浅显，涉及的内容也比较基础。想要彻底搞懂 ThreadLocal 还需要深入剖析源码，还有一些复杂的知识点：例如弱引用等。推荐《码出高效 Java 开发手册》，书中对 ThreadLocal 做了较为详细、深入的讲解，案例易懂，知识点覆盖全面！后续，我会尽可能补充常见关于 ThreadLocal 的知识点。

---

### 弱引用

![Pasted image 20220120090831](https://picbed-sakura.oss-cn-shanghai.aliyuncs.com/notePic/202201220101194.png)
