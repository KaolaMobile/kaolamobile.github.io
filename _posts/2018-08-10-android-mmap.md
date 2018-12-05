---
layout:     post
title:      "Android 高性能日志写入方案"
subtitle:   ""
date:       2018-08-10
author:     "王晨彦"
header-img: "img/bg8.jpg"
tags:
    Android
    mmap 
---

## 前言

网易考拉作为一款超级电商应用，每天都会产生海量日志信息，对日志的写入性能和完整性都有更高的要求。

## 常规方案

Android 中记录日志通常的方式是通过 Java Api 操作文件，当有一条日志要写入的时候，首先，打开文件，然后写入日志，最后关闭文件。使用这种方案虽然当前看上去对程序的影响不大，但是随着日志量的增加，在 Java 中频繁的 IO 操作，容易导致 gc，频繁打开文件，容易引发 CPU 峰值。

下面我们来分析下直接写入文件的流程：

1. 用户发起 write 操作
2. 操作系统查找页缓存<br>
  a.若未命中，则产生缺页异常，然后创建页缓存，将用户传入的内容写入页缓存<br>
  b.若命中，则直接将用户传入的内容写入页缓存
3. 用户 write 调用完成
4. 页被修改后成为脏页，操作系统有两种机制将脏页写回磁盘<br>
  a.用户手动调用 fsync()<br>
  b.由 pdflush 进程定时将脏页写回磁盘

可以看出，数据从程序写入到磁盘的过程中，其实牵涉到两次数据拷贝：一次是用户空间内存拷贝到内核空间的缓存，一次是回写时内核空间的缓存到硬盘的拷贝。当发生回写时也涉及到了内核空间和用户空间频繁切换。

而且相对于机械硬盘，SSD 存储还有一个“写入放大”的问题。这个问题主要和 SSD 存储的物理结构有关。当 SSD 被全部写过一遍之后，再写入的数据是不可以直接更新，只可以通过覆盖重写，在覆盖之前需要先擦除数据。但写入的最小单位是 Page，擦除的最小单位是 Block，而 Block 远大于 Page，所以在写入新数据时就需要先把 Block 上的数据读出来和要写入的数据合并在一起，再把 Block 擦除，最后把读出来的数据重新写入到存储上，这样导致实际写入的数据可能远远大于最开始需要写入的数据。

没想到简单的写文件竟然涉及了这么多操作，只是对于应用层透明而已。

既然每写一次文件会执行这么多次操作，那么我们能不能将日志缓存起来，当达到一定的数量后再一次性的写入磁盘中呢？

这样确实能够大量减少 IO 次数，但是却会引发另一个更严重的问题——**丢日志**

把日志缓存在内存中，当程序发生 Crash 或进程被杀后就无法保证日志的完整性。

一个完善的日志方案，需要满足

- 高效，不能影响系统性能，不能因为引入了日志模块而造成应用卡顿
- 保证日志的完整性，如果不能保证日志完整，那么日志收集就没有意义了
- 如果有必要，对日志进行压缩、加密

## 高性能方案

既然无法减少写入次数，那么我们能不能在写文件的过程中去优化呢？

答案是可以的，使用 **mmap**

mmap 是一种内存映射文件的方法，即将一个文件或者其它对象映射到进程的地址空间，实现文件磁盘地址和进程虚拟地址空间中一段虚拟地址的一一对映关系，函数原型如下

```
#include <sys/mman.h>

void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```

参数说明

![](http://haitao.nosdn3.127.net/1533868671610mmap_params.png) 

mmap 映射模型

![](http://haitao.nosdn4.127.net/1533868650296mmap.png)

示例代码

```
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/mman.h>
main(){
    int fd;
    void *start;
    struct stat sb;
    fd = open("/etc/passwd", O_RDONLY); /*打开/etc/passwd */
    fstat(fd, &sb); /* 取得文件大小 */
    start = mmap(NULL, sb.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
    if(start == MAP_FAILED) /* 判断是否映射成功 */
        return;
    printf("%s", start); munma(start, sb.st_size); /* 解除映射 */
    closed(fd);
}
```

mmap 操作提供了一种机制，让用户程序直接访问设备内存，这种机制，相比较在用户空间和内核空间互相拷贝数据，效率更高。在要求高性能的应用中比较常用。

同时 mmap 能够保证日志的完整性，mmap 的回写时机：

- 内存不足
- 进程退出
- 调用 msync 或者 munmap
- 不设置 MAP_NOSYNC 情况下 30s-60s(仅限FreeBSD)

unmap 函数原型

```
#include <sys/mman.h>

int munmap(void *addr, size_t length);
```

当映射一个文件后，程序就会在 native 内存中申请一块相同大小的空间，因此建议每次映射一小段内容，如 64k，写满后再重新映射文件后面的内容。

有一点需要注意，对于多进程操作文件，使用 Java Api 可以通过 FileLock 同步，而 mmap 不适用于多进程操作同一个文件。对于多进程应用，需要按需映射多个文件。

### 继续优化

根据上述方案，设计 jni 接口，打包 so，引入项目。

考虑到安装包大小，能不能不用 so 呢？

其实 Java 中已经提供了内存映射的实现——**MappedByteBuffer**

MappedByteBuffer 位于 Java NIO 包下，用于将文件内容映射到缓冲区，使用的即是 mmap 技术。通过 FileChannel 的 map 方法可以创建缓冲区

```
MappedByteBuffer raf = new RandomAccessFile(file, "rw");
MappedByteBuffer buffer = raf.getChannel().map(FileChannel.MapMode.READ_WRITE, position, size);
```

有一点比较坑，Java 虽然提供了 map 方法，但是并没有提供 unmap 方法，通过 Google 得知 unmap 方法是有的，不过是私有的

```
// FileChannelImpl.class
private static void unmap(MappedByteBuffer var0) {
    Cleaner var1 = ((DirectBuffer)var0).cleaner();
    if (var1 != null) {
        var1.clean();
    }
}

// Cleaner.class
public void clean() {
    if (remove(this)) {
        try {
            this.thunk.run();
        } catch (final Throwable var2) {
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() {
                    if (System.err != null) {
                        (new Error("Cleaner terminated abnormally", var2)).printStackTrace();
                    }
                    System.exit(1);
                    return null;
                }
            });
        }
    }
}
```

这时我们自然想到了反射调用

```
public static void unmap(MappedByteBuffer buffer) {
    if (buffer == null) {
        return;
    }
    try {
        Class<?> clazz = Class.forName("sun.nio.ch.FileChannelImpl");
        Method m = clazz.getDeclaredMethod("unmap", MappedByteBuffer.class);
        m.setAccessible(true);
        m.invoke(null, buffer);
    } catch (Throwable e) {
        e.printStackTrace();
    }
}
```

> 由于 Android P 已经限制私有 API 的访问，这里仍需要优化以适配 Android P。

为了测试 MappedByteBuffer 的效率，我们把 64byte 的数据分别写入内存、MappedByteBuffer 和磁盘文件 50 万次，并统计耗时

| 方法 | 耗时 |
| -- | -- |
| 内存 | 384ms |
| MappedByteBuffer | 700ms |
| 磁盘文件 | 16805ms |

可以看出 MappedByteBuffer 虽然不及写入内存的性能，但是相比较写入磁盘文件，已经有了质的提升。

## 展望

目前日志模块仅对日志写入性能和完整性提供了保障，对于日志的压缩和加密目前还没有实现，目前缓存的日志都是脱敏数据，后期如果有业务要求安全存储，会考虑添加加密功能。

## 总结

本文主要分析了直接写文件记录日志方式存在的问题，并引申出高性能文件写入方案 mmap，兼顾了写入性能和完整性。最后介绍了内存映射在 Java 层的实现，避免了引入 so。
