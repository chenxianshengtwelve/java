#ThreadLocal源码分析解密
###什么是ThreadLocal
定义：这个类给线程提供了一个本地变量，这个变量是该线程自己拥有的。在该线程存活和ThreadLocal实例能访问的时候,保存了对这个变量副本的引用.当线程消失的时候，所有的本地实例都会被GC。并且建议我们ThreadLocal最好是 private static 修饰的成员
###和Thread的关系
    public void set(T value) {
        Thread t = Thread.currentThread();
        //通过当前线程得到一个ThreadLocalMap
        ThreadLocalMap map = getMap(t);
        //map存在,则把value放入该ThreadLocalMap中
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
	ThreadLocalMap getMap(Thread t) {
        //返回Thread的一个成员变量
        return t.threadLocals;
    }
	void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
	ThreadLocalMap(ThreadLocal firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
Thread和变量放在一个Map->ThreadLocalMap  
Thread类中有一个ThreadLocalMap为null的变量,第一次使用ThreadLocal的时候,我们通过getMAP得到的ThreadLocalMap必然是null,调用createMap方法,在CreatMap中会直接new 一个ThreadLocalMap,里面传入的是当前ThreadLocal.  
小结:Thread里面有一个类似MAP的东西，但是初始化的时候为null，当我们使用ThreadLocal的时候，ThreadLocal会帮助当前线程初始化这个MAP，并且把我们需要和线程绑定的值放入改Map中。map的key为当前ThreadLocal。  
问题：jdk中Entry的key值其实是弱引用的,这代表他将会很快被GC掉。ThreadLocalMap使用ThreadLocal的弱引用作为key，如果一个ThreadLocal没有外部强引用引用他，那么系统gc的时候，这个ThreadLocal势必会被回收，这样一来，ThreadLocalMap中就会出现key为null的Entry，就没有办法访问这些key为null的Entry的value，产生很多人都认为的内存泄露。  
ThreadLocal小片段：jdk给每个ThreadLocal一个标识符以此来区分彼此
###总结
使用线程池的时候，我们应该在改线程使用完该ThreadLocal的时候自觉地调用remove方法清空Entry，这会是一个非常好的习惯。
被废弃了的ThreadLocal所绑定对象的引用，会在以下4情况被清理。

Thread结束时。
当Thread的ThreadLocalMap的threshold超过最大值时。rehash
向Thread的ThreadLocalMap中存放一个ThreadLocal，hash算法没有命中既有Entry,而需要新建一个Entry时。
手工通过ThreadLocal的remove()方法或set(null)。
#ThreadLocal父子线程传递实现方案
###前言
**父线程的准确称呼应该被叫做当前线程的创建线程**。  
1.每一个Thread线程都有属于自己的ThreadLocalMap,里面有一个弱引用的Entry(ThreadLocal,Object),如下
Entry(ThreadLocal k, Object v) {
                super(k);
                value = v;
    }  
2.从ThreadLocal中get值的时候,首先通过Thread.currentThread得到当前线程，然后拿到这个线程的ThreadLocalMap。再传递当前ThreadLocal对象（结合上一点）。取得Entry中的value值  
3.set值的时候同理，更改的是当先线程的ThreadLocalMap中Entry中key为当前Threadlocal对象的value值
###Threadlocal bug？
子线程拿不到父线程的中的ThreadLocal值：
由于ThreadLocal的实现机制,在子线程中get时,我们拿到的Thread对象是当前子线程对象,那么他的ThreadLocalMap是null的,所以我们得到的value也是null。其实很多时候我们是有子线程获得父线程ThreadLocal的需求的，为了解决这种问题，InheritableThreadLocal这个类所做的事情就是为了解决这种问题。
###InheritableThreadLocal实现
public class InheritableThreadLocal<T> extends ThreadLocal<T> {
   
    protected T childValue(T parentValue) {
        return parentValue;
    }

    /**
     * 重写Threadlocal类中的getMap方法，在原Threadlocal中是返回
     *t.theadLocals，而在这么却是返回了inheritableThreadLocals，因为
     * Thread类中也有一个要保存父子传递的变量
     */
    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }

    /**
     * 同理，在创建ThreadLocalMap的时候不是给t.threadlocal赋值
     *而是给inheritableThreadLocals变量赋值
     * 
     */
    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
如果你使用InheritableThreadLocal,那么保存的所有东西都已经不在原来的t.thradLocals里面，而是在一个新的t.inheritableThreadLocals变量中了。  
Q：InheritableThreadLocal是如何实现在子线程中能拿到当前父线程中的值的呢？  
A：一个常见的想法就是把父线程的所有的值都copy到子线程中。  
下面来看看在线程new Thread的时候线程都做了些什么？  

	private void init(ThreadGroup g, Runnable target, String name,
                     long stackSize, AccessControlContext acc) {
       //省略上面部分代码
       if (parent.inheritableThreadLocals != null)
       //这句话的意思大致不就是，copy父线程parent的map，创建一个新的map赋值给当前线程的inheritableThreadLocals。
           this.inheritableThreadLocals =
               ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
      //ignore
    }
而且，在copy过程中是浅拷贝，key和value都是原来的引用地址
大致的解释了一下InheritableThreadLocal为什么能解决父子线程传递Threadlcoal值的问题。  
1.在创建InheritableThreadLocal对象的时候赋值给线程的t.inheritableThreadLocals变量  
2.在创建新线程的时候会check父线程中t.inheritableThreadLocals变量是否为null，如果不为null则copy一份ThradLocalMap到子线程的t.inheritableThreadLocals成员变量中去  
3.因为复写了getMap(Thread)和CreateMap()方法,所以get值得时候，就可以在getMap(t)的时候就会从t.inheritableThreadLocals中拿到map对象，从而实现了可以拿到父线程ThreadLocal中的值
###InheritableThreadLocal还有问题吗？
InheritableThreadLocal不能够解决**线程池**当中获得父线程中ThreadLocal中的值。
由此产生了**transmittable-thread-local**类

