# 误以为是内存泄漏的case

java运行版本

```
java version "1.8.0_121"
Java(TM) SE Runtime Environment (build 1.8.0_121-b13)
Java HotSpot(TM) 64-Bit Server VM (build 25.121-b13, mixed mode)
```

java服务, 长期运行出现的内存泄漏. 运行时间20天左右出现超过系统最大内存, 每天增量在1\~4G之间. 和请求流量存在一定关系

## 基本状况
* jvm内存大小配置, -Xms100g -Xmx100g -XX:MetaspaceSize=1024m -XX:MaxMetaspaceSize=1024m
* gc日志判断, 内存为使用完100g, 平均在30\~50%. 和流量相关
* linux分配内存从100G开始缓慢增长
* springboot框架, 使用tomcat作为容器

## 基本分析
* 堆内存为使用尽, 判断为堆外内存泄漏
* 非JVM原因排查
    * Thread泄漏
    * jni泄漏
    * nio泄漏
* Thread通过启动时间判断可能存在较小的泄漏
    * http-nio-\*-exec 线程在涨
    * elasticsearch 客户端
    * 基本判断不这个泄漏不会产生每日1G\~4G的泄漏
* nio的使用
    * tomcat
    * es客户端
    * hbase客户端
* 已知jni调用
    * gzip

## 使用到的工具
* jstack
* jmap
* pmap
* gdb
* strace
* nmt

## 逐步排查
1. springboot中nio使用堆外内存.看使用未启动堆外内存
    * 使用的tomcat版本 8.5.11.
    * 对应的NioEndpoint配置默认是false
    * 暂未找到如何修改配置, TODO
2. 线程存在膨胀, 但没有出现泄漏的情况
    * 启动和稳定运行线程相差 100\~150个左右, 100个线程也就堆外100M内存
3. 内部使用到了GZIP这个
    * 通过日志分析, 没有报错和异常退出情况, 应该都可以关闭

最后使用java的NMT进行分析
1. 使用NMT进行内存分析
    * 通过crontab定期dump和比较Internal内存一直再涨
    * 初步定为是jvm内部某些机制在申请堆外内存

## 基于java nmt分析

#### nmt使用

jvm启动添加 -XX:NativeMemoryTracking=detail 参数, 通过使用 jcmd pid VM.native_memory 可以获得详细相应信息. 主要几个参数. 详细使用[nmt](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr007.html#BABJGHDB)

我是用到的参数:
* jcmd pid VM.native_memory baseline 创建基线和diff联合使用
* jcmd pid VM.native_memory detail/detail.diff 详情
* jcmd pid VM.native_memory summery/summery.diff 简述

#### 分析过程
1. 通过 jcmd的deatail.diff 发现 InstanceKlass 会在某个时间点涨起来,上涨量可以和概述中 Internal 可以对应上. 根据同事之前判断是 Internal一直上涨, 产生的内存泄漏.
```
-   Internal (reserved=8382506KB +374412KB, committed=8382506KB +374412KB)
        (malloc=8382474KB +374412KB #777683 +46773)
        (mmap: reserved=32KB, committed=32KB)

-----

[0x00007fa360aa9d55] ArrayAllocator<unsigned long, (MemoryType)7>::allocate(unsigned long)+0x175
[0x00007fa360aa863b] BitMap::resize(unsigned long, bool)+0x6b
[0x00007fa360da14ea] OtherRegionsTable::add_reference(void*, int)+0x3ea
[0x00007fa360dbea2f] InstanceKlass::oop_oop_iterate_nv(oopDesc*, FilterOutOfRegionClosure*)+0x12f
                  (malloc=5389560KB +354008KB #673695 +44251)
```
2. 根据 jemalloc生成的内存图, 使用工具[native-jvm-leaks](https://github.com/jeffgriffith/native-jvm-leaks)生成内存创建图, 发现[leak1.svg](./img/leak1.svg). 和diff的内容, 判断堆外内存申请来源于 OtherRegionsTable. 这个数据是G1GC中的一个引用管理的数据结构.
    * G1回收期使用了一种新的数据结构PRT(per region table)来记录对象引用变化(这个变化需要通知RSet进行修改), 分区记录所有引用这的信息, 这个信息存储结构就是PRT. 每个HeapRegion都包含一个PRT, 存储关系 HeapRegion\-\>HeapRegionRemSet\-\>OtherRegionsTable(PRT).
    * OtherRegionsTable 存在三种存储结构. 稀疏(哈希表), 细粒度(PRT指针数组), 粗粒度(以为图表示),
3. 通过这些和网上资料发现这不并不是一个堆外泄漏的case, 只是对g1的部分机制不清导致的. g1为了提高gc的性能, 记录了大量region间的引用关系, 这些数据占用的是堆外内存, 最终导致长时间运行堆外内存不断膨胀的. 当然这个空间是受限的, 只是在我现在服务器上最终会OOM.

## 暂未解决

主要方案:
    1. 调整堆的大小可以适当的减少堆外内存使用
    2. 可以控制JVM参数
      1. -XX:G1RSetRegionEntries PRT可以申请的堆外内存大小, 影响申请次数, 超过次数会改为粗粒度的位图
      2. -XX:G1HeapRegionSize 调整单个Region大小, 太小会导致Region数量过度RSet的占用可能更大.

看这个参数JVM调优是一个平衡的艺术啊, 还是默认吧, 有限考虑优化代码. 接手的破服务直接配置了100G内存, 根据gc日志判断根本使用不到(观察最大才70G),  cache之间的数据存在互相引用, 导致RSet存储了大量的互相引用, 还存在cache中元素字段会动态修改, 对代码进行调整梳理和优化可能受益会比较大, 减少混乱引用, 对代码还有很多优化空间, 优先改代码效果可能会更好.

## G1GC

G1GC将堆内存分为多个区域(Region), 每个区大小固定在1M\~32M, 又启动时决定. 之前的gc算法将内存主要分为两个大区\-\-老生代和新生代. g1虽然也分新老生代, 但区域变多为了提高gc的效率, 为每个区域配置了RSet存储跨区域之间的引用.

引用关系记录主要有两种, PrintIn, PrintOut, 当然互相都记录也可以. G1采用的是PointIn方式, 对应每个region存在的RSet中记录所有指向它的region区域信息
```
例:obj1.field1=obj2，
PointOut方式:则在obj1所在region的RSet记录obj2的位置
PointIn方式:则在obj2所在region记录obj1的位置
```

区域间引用关系共有五种：
* 分区内引用
* 新生代分区Y1引用新生代分区Y2
* 新生代分区Y1引用老年代分区O1
* 老年代分区O1引用新生代分区Y1
* 老年代分区O1引用老年代分区O2

YGC时, GC root主要是两类:栈空间和老年代分区到新生代分区的引用关系. Mixed GC时, 由于仅回收部分老年代分区, 老年代分区之间的引用关系也将被使用. 因此, 只需要记录两种引用关系: 老年代分区引用新生代分区, 老年代分区之间的引用.

为了节省空间, G1采用了三种等级进行RSet的存储, 分别是稀疏表, 细粒度PerRegionTable, 粗粒度位图.

// TODO


#### 堆外内存计算

[源码地址](http://hg.openjdk.java.net/), 我下载的jdk8u的hotspot, 下载巨慢, 慢慢下大概10M左右. 大学毕业后第一次认真去看c++的代码, 看的真是费劲啊, c++语法特性太多了. g1的代码在share/vm/gc_implementation/g1/目录下.

InstanceKlass::oop_oop_iterate_nv 没有找到对应代码, 向下找 InstanceKlass::oop_oop_iterate_nv::add_reference找到了, 在heapRegionRemSet.cpp内. 相关g1运行参数在 heapRegion.cpp文件内.

根据代码大量堆外内存申请是HeapRegionRemSet中OtherRegionsTable的PerRegionTable申请的, 这个结构是细粒度的一个card表, 有bitmap存储. PerRegionTable有一个BitMap的结构, 这部分会申请堆外的内存, 根据追代码判断每个bitmap大小根据CardsPerRegion确定, 在没有超过阈值的情况下可以不断的申请. 个数根据\_max\_fine\_entries进行确定. 这个PerRegionTable结构是每个region都存在一个的, 这里可能会申请大量堆外内存. 这可以和之前统计数据对应上.

不知道cmd的日志为什么不把这部分内存算作GC的, 而是算作内部的.

堆外内存的使用时根据可以根据\_max\_fine\_entries\*CardsPerRegion\*region数量决定的.一下是计算相关代码

**G1初始参数计算**

```C++
常量:
HeapRegionBounds::min_size = 1024*1024
HeapRegionBounds::target_number = 2048
HeapRegionBounds::max_size = 32*1024*1024
LogHeapWordSize 和机器有关 2或者3 根据_LP64这个参数决定
CardTableModRefBS::card_shift = 9

void HeapRegion::setup_heap_region_size(size_t initial_heap_size, size_t max_heap_size) {
    uintx region_size = G1HeapRegionSize;
    if (FLAG_IS_DEFAULT(G1HeapRegionSize)) {
      size_t average_heap_size = (initial_heap_size + max_heap_size) / 2;
      region_size = MAX2(average_heap_size / HeapRegionBounds::target_number(),
                         (uintx) HeapRegionBounds::min_size());
    }

    int region_size_log = log2_long((jlong) region_size);
    // Recalculate the region size to make sure it's a power of
    // 2. This means that region_size is the largest power of 2 that's
    // <= what we've calculated so far.
    region_size = ((uintx)1 << region_size_log);

    // Now make sure that we don't go over or under our limits.
    if (region_size < HeapRegionBounds::min_size()) {
      region_size = HeapRegionBounds::min_size();
    } else if (region_size > HeapRegionBounds::max_size()) {
      region_size = HeapRegionBounds::max_size();
    }

    // And recalculate the log.
    region_size_log = log2_long((jlong) region_size);

    // Now, set up the globals.
    guarantee(LogOfHRGrainBytes == 0, "we should only set it once");
    LogOfHRGrainBytes = region_size_log;

    guarantee(LogOfHRGrainWords == 0, "we should only set it once");
    LogOfHRGrainWords = LogOfHRGrainBytes - LogHeapWordSize;

    guarantee(GrainBytes == 0, "we should only set it once");
    // The cast to int is safe, given that we've bounded region_size by
    // MIN_REGION_SIZE and MAX_REGION_SIZE.
    GrainBytes = (size_t)region_size;

    guarantee(GrainWords == 0, "we should only set it once");
    GrainWords = GrainBytes >> LogHeapWordSize;
    guarantee((size_t) 1 << LogOfHRGrainWords == GrainWords, "sanity");

    guarantee(CardsPerRegion == 0, "we should only set it once");
    CardsPerRegion = GrainBytes >> CardTableModRefBS::card_shift;
}
```

基于以上代码和当前配置计算出, region_size大小是32M, region数量在3200个左右, CardsPerRegion是8Kb, region_size_log是25

**\_max\_fine\_entries 计算**

```c++
G1RSetRegionEntries=G1RSetRegionEntriesBase * (region_size_log_mb + 1);
G1RSetRegionEntriesBase=256

// G1RSetRegionEntries 计算
void HeapRegionRemSet::setup_remset_size() {
  // Setup sparse and fine-grain tables sizes.
  // table_size = base * (log(region_size / 1M) + 1)
  const int LOG_M = 20;
  int region_size_log_mb = MAX2(HeapRegion::LogOfHRGrainBytes - LOG_M, 0);
  if (FLAG_IS_DEFAULT(G1RSetSparseRegionEntries)) {
    G1RSetSparseRegionEntries = G1RSetSparseRegionEntriesBase * (region_size_log_mb + 1);
  }
  if (FLAG_IS_DEFAULT(G1RSetRegionEntries)) {
    G1RSetRegionEntries = G1RSetRegionEntriesBase * (region_size_log_mb + 1);
  }
  guarantee(G1RSetSparseRegionEntries > 0 && G1RSetRegionEntries > 0 , "Sanity");
}

// 针对_max_fine_entries的计算
  if (_max_fine_entries == 0) {
    assert(_mod_max_fine_entries_mask == 0, "Both or none.");
    size_t max_entries_log = (size_t)log2_long((jlong)G1RSetRegionEntries);
    _max_fine_entries = (size_t)1 << max_entries_log;
    _mod_max_fine_entries_mask = _max_fine_entries - 1;

    assert(_fine_eviction_sample_size == 0
           && _fine_eviction_stride == 0, "All init at same time.");
    _fine_eviction_sample_size = MAX2((size_t)4, max_entries_log);
    _fine_eviction_stride = _max_fine_entries / _fine_eviction_sample_size;
  }
```
通过计算可以得知\_max\_fine\_entries=1024, 基于之前计算总共堆外内存可以申请最大值在25G, 而整个系统内存共125G, 堆内存100G迟早会被linux系统kill掉.



## 参考
https://blog.csdn.net/jicahoo/article/details/50933469

https://github.com/jeffgriffith/native-jvm-leaks

https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr007.html#BABJGHDB

[一个总结, 国外](https://medium.com/@milan.mimica/everybody-leaks-f210631f13ef)

[相似的问题](https://github.com/prestodb/presto/issues/9553)

[JVM G1源码分析和调优](https://book.douban.com/subject/33408230/)

[JVM G1 源码分析（三）- Remembered Sets](https://blog.csdn.net/a860MHz/article/details/97276211)
