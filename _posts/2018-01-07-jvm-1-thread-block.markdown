---
layout: post
title: 深入理解JVM(1)——线程安全
date: 2018-01-07 00:00:00 +0300
description: You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. # Add post description (optional)
img: fantuan-1.jpg # Add image post (optional)
tags: [Jvm] # add tag
---

&emsp;&emsp;*最近在工作中一直在处理关于Java虚拟机的相关问题，恰逢在拜读周志明老师所著的《深入理解Java虚拟机》。于是准备根据切身相关的知识短板，整理一份学习笔记，站在巨人的肩膀上提炼一些心得。系列文章不参与任何形式的盈利活动，谢绝转载。*

### 线程安全

&emsp;&emsp;在这里我们不再讨论线程安全的具体定义，而是由安全性从强到弱去陈列5种线程安全的操作分类：不可变、绝对线程安全、相对线程安全、线程兼容和线程对立。

* 不可变

&emsp;&emsp;在JDK1.5对内存模型修正之后，有一个恒定的真理就是不可变（Immutable）的对象一定是线程安全的，无论是对象的实现方法还是方法的调用者，都不需要再采取任何的线程安全的。“不可变”保证了对象行为不会影响自己的状态，例如对于一个java.lang.String类型对象，我们调用它的substring()、replace()等等这些方法，不会影响它原来的值，只会返回一个新构造的字符串对象。

&emsp;&emsp;我们不妨从String类的源码一探究竟：

{% highlight java %}
//用于存储字符串
private final char value[];

//缓存String的hash值
private int hash; // Default to 0

//use serialVersionUID from JDK 1.0.2 for interoperability
private static final long serialVersionUID = -6849794470754667710L;

public String() {
    this.value = new char[0];
}

//参数为String类型
public String(String original) {
    this.value = original.value;
    this.hash = original.hash;
}
{% endhighlight %}

&emsp;&emsp;可以看到String中包含了一个不可变的char数组用来存放字符串，一个int型的变量hash来存放计算后的hash值。字符串常量池是Java方法区中一个特殊的存储区域, 当创建一个String对象时,假如此字符串值已经存在于常量池中,则不会创建一个新的对象,而是引用已经存在的对象。这也从另一个侧面反映了String为一个不可变类的重要性。

&emsp;&emsp;那么如何实现一个不可变类呢？<br />

&emsp;&emsp;1.将类声明为final，所以它不能被继承。<br />
&emsp;&emsp;3.对变量不要提供setter方法。<br />
&emsp;&emsp;4.将所有可变的成员声明为final，这样只能对它们赋值一次。<br />
&emsp;&emsp;5.通过构造器初始化所有成员，进行深拷贝(deep copy)。<br />
&emsp;&emsp;6.在getter方法中，不要直接返回对象本身，而是克隆对象，并返回对象的拷贝。

* 绝对线程安全

&emsp;&emsp;绝对的线程安全需要满足“不管运行时环境如何，调用者不需要任何额外的同步措施”。需要注意的是，Java API中有很多类标注自己是线程安全的，而这种线程安全并不是绝对的线程安全，我们把原因放在下一小节“相对线程安全”中解释。

* 相对线程安全

&emsp;&emsp;实际上我们通常意义上所讲的线程安全指的是相对线程安全，它保证了对这个对象单独的操作是线程安全的，我们在调用的时候不需要做额外的保障措施，但是对于一些特定顺序的连续调用，就可能需要在调用端使用额外的同步手段来保证调用的正确性。我们来看java.util.Vector这个“线程安全”的容器。（示例节选自书中）

{% highlight java %}
private static Vector<Integer> vector = new Vector<Integer>();

public static void main(String[] args) {
    while (true) {
        for (int i = 0; i < 10; i++) {
            vector.add(i);
        }

        Thread removeThread = new Thread(new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < vector.size(); i++) {
                    vector.remove(i);
                }
            }
        });

        Thread printThread = new Thread(new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < vector.size(); i++) {
                    System.out.println(vector.get(i));
                }
            }
        });

        removeThread.start();
        printThread.start();

        //不要同时产生过多的线程，否则会导致操作系统假死
        while (Thread.activeCount() > 20);
    }
}
{% endhighlight %}

&emsp;&emsp;运行这个main方法后会看到报错“Exception in thread  “Thread-132” java.lang.ArrayIndexOutOfBoundException:...”。很明显，尽管我们使用的Vector的get(),remove(),size()方法都是同步的，但是在多线程环境中，如果不在方法调用端做额外的同步措施的话，使用这段代码仍是不安全的。

&emsp;&emsp;为了解决这里的线程不安全问题，我们可以在removeThread和printThread定义中将run()方法中的for循环代码块用synchronized关键字修饰，以防止原先代码中“不可预知”的错误。

* 线程兼容

&emsp;&emsp;线程兼容是指对象本身并不是线程安全的，但是可以通过在调用端正确的使用同步手段来保证对象在并发环境中可以安全地使用，需要注意的是，Java API中大部分的类都是属于线程兼容的。个人也认为，考验一个编程人员是否真的有线程安全意识，也是通过这类线程兼容的类体现的。

* 线程对立

&emsp;&emsp;线程对立是指无论调用端师傅采取了同步措施，都无法在多线程环境中并发使用的代码。这类情况实际上十分少见，感兴趣的读者朋友可以了解JDK中已经声明废弃的两个方法suspend()和resume()，这两种线程对立的方法会存在相当程度的死锁风险。

### 小节

&emsp;&emsp;本文介绍了线程安全所涉及的概念和分类，实际上我认为了解并发的底层实现才是真正的关键所在，这和操作系统的知识是紧紧关联的，这个点我们留在后面的章节讨论。
