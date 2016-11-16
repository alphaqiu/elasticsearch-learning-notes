# Elasticsearch 启动时检查项

## 检查 Java 堆大小

初始堆大小 ＝ 最大堆大小

`/etc/elasticsearch/jvm.options`

`-Xms16g -Xmx16g`

## 检查文件描述符

Elasticsearch 会打开非常多的文件句柄，例如 每个分片上的segments，以及其他文件。

临时更改系统设置： `ulimit -n 65536`

永久更改系统设置：`/etc/security/limits.conf`

## **检查内存锁定**

JVM 的垃圾收集器会访问堆栈上的每一页。如果任何一页被交换出了内存到磁盘，它们必须被重新交换到内存中。Elasticsearch 频繁的服务请求会造成大量的磁盘垃圾碎片。有若干中方法配置系统来禁止磁盘交换。其中一个办法是通过`mlockall`锁定 JVM 堆栈内存。这是通过设置 Elasticsearch 的配置 [bootstrap.memory_lock](#bootstrap.memory_lock)。但是要使 mlockall 生效，需要设置服务器参数。

设置系统内存限制的几种方式：

- 临时设置 `ulimit -l unlimited`


- 永久设置 设置 `/etc/security/limits.conf` memlock unlimited
- systemd 要设置 `LimitMEMLOCK=infinity`

## **检查最大线程数**

Elasticsearch 会将执行的请求分拆成不同的阶段，并将这些阶段转换到不同的线程池去执行。

Elasticsearch 需要使用至少 2048 线程数。

`/etc/security/limit.conf`文件中的`nproc`设置成至少该数值。

## **检查最大虚拟内存**

Elasticsearch 和 Lucene 使用 mmap 管理索引地址空间，mmap 脱离 JVM 堆栈并保存着具体的索引数据，但是在内存中又可以很快被访问到。需要无限制的虚拟内存。

`/etc/security/limit.conf`文件中的 `as`设置成`unlimited`

## **检查最大map count**

接上一条提到的，Elasticsearch 也需要创建大量的内存映射区域的能力。该项检查将检查linux 内核是否允许处理至少262144内存映射区域（只针对linux）。

`sysctl vm.max_map_count=262144` 至少设置 262144

## 检查 Client JVM

这个jvm.options 里面已经设置好了， 服务器模式。

## 检查是否使用了序列化垃圾收集器

检查垃圾收集器。序列化垃圾收集器仅适用于单核的CPU以及极小堆。使用它会影响性能。这个在 jvm.options 内也设置好了。要使用 CMS Collector

## 检查 OnError 和 OnOutOfMemoryError

JVM 设置项 允许设置 OnError 和 OnOutOfMemoryError 过滤器，去触发任意脚本。

但是 Elasticsearch 系统会调用 seccomp 过滤器且，该过滤器禁止 forking

> Seccomp(secure computing)是Linux kernel （自从2.6.23版本之后）所支持的一种简洁的sandboxing机制。它能使一个进程进入到一种“安全”运行模式，该模式下的进程只能调用4种系统调用（system calls），即read(), write(), exit()和sigreturn()，否则进程便会被终止。

因此 OnError 和 OnOutOfMemoryError 过滤器与之不兼容。

instead, upgrade to Java 8u92 and use the JVM flag `ExitOnOutOfMemoryError`. While this does not have the full capabilities of `OnError` nor `OnOutOfMemoryError`, arbitrary forking will not be supported with seccomp enabled.