CAS
CAS是英文单词CompareAndSwap的缩写，即比较并替换，是一种实现并发算法试尝用到的技术。CAS需要有3个操作数：内存地址V，旧的预期值A，即将要更新的目标值B。

示例代码：
public class VolatileTest {
    static int data = 0;
    public static void main(String[] args) {
        IntStream.range(0, 2).forEach((i) -> {
            new Thread(() -> {
                try {
                    Thread.sleep(20);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                IntStream.range(0, 100).forEach(y -> {
                    data++;
                });
            }).start();

        });
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(data);
    }
}
测试结果：
> Task :VolatileTest.main()
> 104
> 结果分析：上述代码线程1，和线程2分别对data自增，结果在IDEA上运行每次的结果都不是200，这是由于两个线程同时操作了共享变量导致结果不正常，即是线程不安全。

解决方案一：使用synchronized对自增代码进行加锁
//类中定义Object对象，创建一个锁对象
static Object lock = new Object();
IntStream.range(0, 100).forEach(y -> {
    synchronized (lock.getClass()) {
        data++;
    }
});
测试结果：
> Task :VolatileTest.main()
> 200
> 结果分析：对自增代码加synchronized关键字，县城内使用同步代码块，由JVM自身的机制来保障线程的安全，缺点是效率比较慢。

使用Lock锁
解决方案二：使用Lock锁
//定义一个lock，用lock自带的加锁和解锁操作，来保证线程安全
static ReentrantLock reentrantLock = new ReentrantLock();

reentrantLock.lock();
data++;
reentrantLock.unlock();
执行结果：
> Task :VolatileTest.main()
> 200
> 结果分析：高并发场景下，使用Lock锁要比使用synchronized关键字新能就得到极大提高，Lock底层是通过AQS+CAS机制来实现，缺点是一旦unlock()方法使用不规范，可能导致死锁问题。

解决方案三：使用Atomic原子类实现自增
static AtomicInteger atomicData = new AtomicInteger();
atomicData.incrementAndGet();
执行结果：
> Task :VolatileTest.main()
> 200
> 结果分析：在并发量不大的情况下，AtomicInteger比Lock更加安全，性能也能接受。通过JVM底层机制来保证，自动释放锁，无需手动释放。

解决方案四：使用LongAdder原子类，实现方式和方案三类似
//成员变量
static LongAdder longAdderData = new LongAdder();
longAdderData.increment();
执行结果：
> Task :VolatileTest.main()
> 200
> 结果分析：LongAdder是JDK1.8中新增的类，和AtomicInteger原子类相似，都是java.util.concurrent.atomic 并发包下的。LongAdder适合高并发长江，特别是写大于读的场景，代价是消耗更多空间。

CAS原理分析：
本次着重由方案三中的AtomicInteger原子类如手，来分析CAS

打开AtomicInteger的incrementAndGet()方法，该方法调用的Unsafr类中的compareAndSwapInt()方法，compareAndSwapInt()方法使用native方法修饰，表示在底层使用C++写的
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
}
注释：compareAndSwapInt的具体实现，cmpxchg： compare and exchange的缩写
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x))
  UnsafeWrapper("Unsafe_CompareAndSwapInt");
  oop p = JNIHandles::resolve(obj);
  jint* addr = (jint *) index_oop_from_field_offset_long(p, offset);
  return (jint)(Atomic::cmpxchg(x, addr, e)) == e;
UNSAFE_END

注释：cmpxchg(x, addr, e)的具体实现
inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {
  int mp = os::is_MP();
  __asm__ volatile (LOCK_IF_MP(%4) "cmpxchgl %1,(%3)"   //asm汇编语言  cmpxchgl：compare and exchange 比较并交换   IF_MP：if multiprocessors 如果是多个CPU，前面加上了lock指令
                    : "=a" (exchange_value)
                    : "r" (exchange_value), "a" (compare_value), "r" (dest), "r" (mp)
                    : "cc", "memory");
  return exchange_value;
}

注释：is_MP()底层代码具体实现
static inline bool is_MP() {
    // During bootstrap if _processor_count is not yet initialized
    // we claim to be MP as that is safest. If any platform has a
    // stub generator that might be triggered in this phase and for
    // which being declared MP when in fact not, is a problem - then
    // the bootstrap routine for the stub generator needs to check
    // the processor count directly and leave the bootstrap routine
    // in place until called after initialization has ocurred.
    return (_processor_count != 1) || AssumeMP;
 }

 注释：最终实现
#define LOCK_IF_MP(mp) "cmp $0, " #mp "; je 1f; lock; 1: "
//cmpxchg = cas修改变量值
lock cmpxchg 指令（cmpxchg指令本身是非原子性，但是加了lock保证了cmpxchg原子性）