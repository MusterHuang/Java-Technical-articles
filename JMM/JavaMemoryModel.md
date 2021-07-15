**Java内存模型**

一个普通Java对象在内存中的布局分为三个个部分，分别是***header(对象头),instance data（实例数据）,padding（对齐填充）***。
1.header:对象头用于存储对象的元信息
  对象头又分为两部分：
    a.markword:8Bytes,用于存储对象自身的运行时数据，标记锁信息、GC信息、IdentityHashCode等，如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等，
      这部分数据的长度在32位和64位的虚拟机中分别位32bit和64bit.
    b.class pointer：指向它的类元数据的指针，用于判断对象属于哪个类的实例。
      另外，如果对像是一个数组，那在对象头中还必须有一块用于记录数组长度的数据，因为虚拟机可以通过普通Java对象的元数据信息确定Java对象的大小，但是从数组的元数据中却无法确定数组的大小。
      JVM开启内存压缩则指针大小为4个字节，不开启则是8个字节，JVM默认是开启的。
      *cmd命令可以查看参数信息：
      java -XX:+PrintCommandLineFlags -version
      -XX:InitialHeapSize=264183232 -XX:MaxHeapSize=4226931712 -XX:+PrintCommandLineFlags -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:-UseLargePagesIndividualAllocation -XX:+UseParallelGC
      java version "1.8.0_201"
      Java(TM) SE Runtime Environment (build 1.8.0_201-b09)
      Java HotSpot(TM) 64-Bit Server VM (build 25.201-b09, mixed mode)
      其中UseCompressedClassPointers是压缩指针，默认开启。
      其中UseCompressedOops（ordinaty object pointer 普通对象指针）可以看到64-Bit Server系统是64位的，所以一个指针默认是8个字节，当开启压缩指针的时候会把指针压缩成4个字节。
 **2.instance data**:实例数据部分是对象真正存储的有效信息，也是在程序代码中所定义各种类型的字段内容。无论是从父类继承下来的，还是在子类中定义的，都需要记录下来。父类定义的变量会出现在子类定义的变量的前面。
      各字段的分配策略为longs/doubles、ints、shorts/chars、bytes/boolean、oops(ordinary object pointers)，相同宽度的字段总是被分配到一起，便于之后取数据。
      基本类型 (int，float 4字节 long，double 8字节，char，short 2字节，byte 1字节 ，boolean 实际大小1bit  占用1字节)
      instance data  的总大小如果不能被4整除，那么会补齐 (alignment/padding gap)
 **3.padding**:对齐填充并不是必然存在的，也没有特别的含义，它仅仅起着占位符的作用。为什么需要有对齐填充呢？由于hotspot VM的自动内存管理系统要求对象起始地址必须是8字节的整数倍，换句话，就是对象的大小必须是8字节的整数倍。
   而对象头正好是8字节的倍数。因此，当对象实例数据部分没有对齐时，就需要通过对齐填充来补全。
   补齐分为两种情况：
   a.若是没有引用，则直接判断是否被8整除,不足的用padding对齐。
   b.若是存在引用 ，则将引用对应的4字节加上，然后判断是否被8整除,不足的用padding对齐。

工具：JOL = Java Object Layout
<dependencies>
    <!-- https://mvnrepository.com/artifact/org.openjdk.jol/jol-core -->
    <dependency>
        <groupId>org.openjdk.jol</groupId>
        <artifactId>jol-core</artifactId>
        <version>0.9</version>
    </dependency>
</dependencies>

代码示例：
Object o = new Object();
System.out.println(ClassLayout.parseInstance(o).toPrintable());

输出：
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
问题：Object o = new Object()在内存中所占的字节？Object o = new User(int id, String name);
答：1.开启压缩16个字节，没有开启压缩也是16个字节。 2.24个字节，8(markword) + 4（pointer） + 4(int) + 4(String 引用类型+4) + 4(padding) = 24个字节。


数组对象：
示例代码：
int[] a = {1,2,3}; //int[] a = new int[3];效果一样
System.out.println(ClassLayout.parseInstance(a).toPrintable());

输出：
[I object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           6d 01 00 f8 (01101101 00000001 00000000 11111000) (-134217363)
     12     4        (object header)                           03 00 00 00 (00000011 00000000 00000000 00000000) (3)
     16    12    int [I.<elements>                             N/A
     28     4        (loss due to the next object alignment)
Instance size: 32 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

可以看到数组对象的布局和普通对象有差异，对象a在内存中站的字节数为24，由8（mark word）+ 4（klass pointer） + 4(arrat length) + 12(3个int类型的值) + 4（padding）