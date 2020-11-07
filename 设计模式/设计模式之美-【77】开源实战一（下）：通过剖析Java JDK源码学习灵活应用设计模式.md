# 模版模式在Collections类中的应用

> 策略、模版、职责链三个模式常用在框架的设计中，提供框架的扩展点。让框架使用者，在不修改框架源码的情况下，基于扩展点定制框架的功能。`Java` 中的Collections类的sort()函数就是利用了模版模式的这个扩展特性
>
> Collections.sort()实现了对集合的排序。为了扩展性，它将其中“比较大小”这部分逻辑，委派给用户来实现。如果我们把比较大小这部分逻辑看作整个排序逻辑的其中一个步骤，那我们就可以把它看作模版模式。不过，从代码实现的角度来看，它看起来有点类似之前讲过的JdbcTemplate，并不是模版模式的经典代码实现，而是基于Callback回调机制来实现的。
>
> 不过，在其他资料中，有人说，Collections.sort()使用的是策略模式。这样的说法也不是没有道理。如果我们并不把“比较大小”看作排序逻辑中的一个步骤，而是看作一种算法或者策略，那我们就可以把它看作一种策略模式的应用。
>
> 不过，这不是典型的策略模式，前面讲到，在典型的策略模式中，策略模式分为策略的定义、创建、使用这三部分。策略通过工厂模式来创建，并且在程序运行期间，根据配置、用户输入、计算结果等这些不确定因素，动态决定使用哪种策略。而在Collections.sort()函数中，策略的创建并非通过工厂模式，策略的使用也非动态确定。

# 观察者模式在JDK中的应用

> Google Guava 的EventBus框架，提供了观察者模式的骨架代码。使用了EventBus，不需要从零开始开发观察者模式。实际上，Java JDK也提供了观察者模式的简单框架实现。在平时的开发中，如果不希望引入Google Guava开发库，可以直接使用Java语言本身提供的这个框架类。
>
> 它比EventBus要简单多了，只包含两个类：java.util.Observable和java.util.Observer。前者是被观察者，后者是观察者。他们的代码实现也非常简单：
>
> ```java
> 
> public interface Observer {
>     void update(Observable o, Object arg);
> }
> 
> public class Observable {
>     private boolean changed = false;
>     private Vector<Observer> obs;
> 
>     public Observable() {
>         obs = new Vector<>();
>     }
> 
>     public synchronized void addObserver(Observer o) {
>         if (o == null)
>             throw new NullPointerException();
>         if (!obs.contains(o)) {
>             obs.addElement(o);
>         }
>     }
> 
>     public synchronized void deleteObserver(Observer o) {
>         obs.removeElement(o);
>     }
> 
>     public void notifyObservers() {
>         notifyObservers(null);
>     }
> 
>     public void notifyObservers(Object arg) {
>         Object[] arrLocal;
> 
>         synchronized (this) {
>             if (!changed)
>                 return;
>             arrLocal = obs.toArray();
>             clearChanged();
>         }
> 
>         for (int i = arrLocal.length-1; i>=0; i--)
>             ((Observer)arrLocal[i]).update(this, arg);
>     }
> 
>     public synchronized void deleteObservers() {
>         obs.removeAllElements();
>     }
> 
>     protected synchronized void setChanged() {
>         changed = true;
>     }
> 
>     protected synchronized void clearChanged() {
>         changed = false;
>     }
> }
> ```

> 先来看changed成员变量
>
> > 表明被观察者有没有状态更新
>
> 再来看notifyObservers()函数
>
> > ～
>
> 单例模式在Runtime类中的应用
>
> > java.lang.Runtime类是一个单例类，实现了最简单的饿汉式的单例实现方式
>
> 每个Java应用在运行时会启动一个JVM进程，每个JVM进程都只对应一个Runtime实例，用于查看JVM状态以及控制JVM行为。进程内唯一，所以比较适合设计为单例。在编程的时候，不能自己去实例化一个Runtime对象，只能通过getRuntime()静态方法来获得。
>
> vector单线程安全，有synchronized加强，但是多线程情况下并非安全的，原子性问题？