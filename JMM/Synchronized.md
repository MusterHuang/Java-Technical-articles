Synchronized

示例代码：
Object o = new Object();
System.out.println(ClassLayout.parseInstance(o).toPrintable());
synchronized (o) {
    System.out.println(ClassLayout.parseInstance(o).toPrintable());
}

运行结果：
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           b8 f7 62 02 (10111000 11110111 01100010 00000010) (40040376)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

结果分析：
可以看到加锁和没加锁在内存布局上可以看出差距，加锁对象的mark word的value值和没有加锁的不一致，由此可知对象是否加锁的信息在mark down里边记录