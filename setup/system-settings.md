# 重要的系统设置项

理想情况下，一台服务器部署一套 Elasticserarch 并使用服务器全部可用资源。所以，需要设置服务器配置。

下列给出的清单是必须在生产环境中要处理的：

- 设置 JVM 堆大小
- 禁止 swap 交换
- 提高文件句柄数
- 确保充足的虚拟内存
- 确保充足的线程数

默认情况下，Elasticsearch 假定当前是工作在开发模式，如果以上清单任何一项设置没有配置正确，它会打印警告级别的日志，但仍然可以启动它。

一旦设置好 Elasticsearch 配置文件中的网络相关设置，诸如`network.host`，它会假定你将要转换到生产环境，并且以上提到的警告将提升为Exceptions。这些错误将阻止启动。

## 配置系统参数

设置临时生效：

```bash
# 设置最大打开的文件句柄数
ulimit -n 65536
# 设置内存锁定限制
ulimit -l unlimited
# 设置线程数
ulimit -u 2048
# 调整系统内核最大map映射数(虚拟内存)
sysctl -w vm.max_map_count=262144
```

设置永久生效：

`/etc/security/limits.conf`

```bash
#<domain> can be:
#    - a user name
#    - a group name, with @group syntax
#    - the wildcard *, for default entry
#    - the wildcard %, can be also used with %group syntax,
#         for maxlogin limit
#
#<type> can have the two values:
#   - "soft" for enforcing the soft limits
#   - "hard" for enforcing hard limits
#<item> can be one of the following:
#  - nofile - max number of open files
#  - memlock - max locked-in-memory address space (KB)
#  - nproc - max number of processes
#  - as - address space limit (KB)
#<domain>      <type>  <item>         <value>
elasticsearch - nofile  65536
elasticsearch - memlock unlimit
elasticsearch - nproc   unlimit
```

`/etc/sysctl.conf`

```bash
vm.max_map_count = 262144
```

**Systemd 设置**

参考`/usr/lib/systemd/system/elasticsearch.service`

```bash
[Unit]
Description=Elasticsearch
Documentation=http://www.elastic.co
Wants=network-online.target
After=network-online.target

[Service]
Environment=ES_HOME=/usr/share/elasticsearch
Environment=CONF_DIR=/etc/elasticsearch
Environment=DATA_DIR=/var/lib/elasticsearch
Environment=LOG_DIR=/var/log/elasticsearch
Environment=PID_DIR=/var/run/elasticsearch
EnvironmentFile=-/etc/sysconfig/elasticsearch

WorkingDirectory=/usr/share/elasticsearch

User=elasticsearch
Group=elasticsearch

ExecStartPre=/usr/share/elasticsearch/bin/elasticsearch-systemd-pre-exec

ExecStart=/usr/share/elasticsearch/bin/elasticsearch \
                                                -p ${PID_DIR}/elasticsearch.pid \
                                                --quiet \
                                                -Edefault.path.logs=${LOG_DIR} \
                                                -Edefault.path.data=${DATA_DIR} \
                                                -Edefault.path.conf=${CONF_DIR}

# StandardOutput is configured to redirect to journalctl since
# some error messages may be logged in standard output before
# elasticsearch logging system is initialized. Elasticsearch
# stores its logs in /var/log/elasticsearch and does not use
# journalctl by default. If you also want to enable journalctl
# logging, you can simply remove the "quiet" option from ExecStart.
StandardOutput=journal
StandardError=inherit

# Specifies the maximum file descriptor number that can be opened by this process
LimitNOFILE=65536

# Specifies the maximum number of bytes of memory that may be locked into RAM
# Set to "infinity" if you use the 'bootstrap.memory_lock: true' option
# in elasticsearch.yml and 'MAX_LOCKED_MEMORY=unlimited' in /etc/sysconfig/elasticsearch
#LimitMEMLOCK=infinity

# Disable timeout logic and wait until process is stopped
TimeoutStopSec=0

# SIGTERM signal is used to stop the Java process
KillSignal=SIGTERM

# Java process is never killed
SendSIGKILL=no

# When a JVM receives a SIGTERM signal it exits with code 143
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target

# Built for distribution-5.0.0 (distribution)
```

要打开`LimitMEMLOCK=infinity`

如果是运行于 docker 镜像，需要设置 `/usr/lib/systemd/system/docker.service` ，增加：

```bash
LimitMEMLOCK=infinity
```

设置好后，执行

```bash
systemctl daemon-reload
# running on the server
systemctl restart elasticsearch.service
# or running in docker
systemctl restart docker
```

**设置 JVM 系统属性**

配置文件在`/etc/elasticsearch/jvm.options`

另一种可选的方式是设置环境变量`ES_JAVA_OPTS`

```bash
export ES_JAVA_OPTS="$ES_JAVA_OPTS -Djava.io.tmpdir=/path/to/temp/dir"
./bin/elasticsearch
```

## 配置 JVM 堆大小

默认最小堆和最大堆大小为2GB。对于任何一个业务部署来说，这个都太小了。如果你正在使用这些默认堆内存配置，你的集群配置可能有点问题。对于堆大小设置的原则如下：

- 设置最小堆（Xms）和最大堆（Xmx）大小一致。

- 可用堆越大，可缓存的空间越多。不过更大的堆也会延长垃圾收集时暂停的时间。

- Xmx 不能超过物理内存的 50%，剩余物理内存需要留给系统缓存内核文件。

- Xmx 不要超过 32GB，jvm 会对超过32GB的内存使用压缩指针技术。通过观察日志查看以下内容来检查该项。

  ```
  heap size [1.9gb], compressed ordinary object pointers [true]
  ```

- 最好堆大小设置在零基对象压缩指针（zero-based compressed oops）阀值以下，不同的系统阀值不同，大多数系统安全阀值在26GB，有些系统可以设置到30GB。可以通过以下方式验证系统零基对象压缩指针的阀值：设置 jvm.options 参数，

  `-XX:+UnlockDiagnosticVMOptions -XX:+PrintCompressedOopsMode`

  并观察日志：

  ```
  heap address: 0x000000011be00000, size: 27648 MB, zero based Compressed Oops
  ```

  以上零基对象压缩指针替换成以下：

  ```
  heap address: 0x0000000118400000, size: 28672 MB, Compressed Oops with base: 0x00000001183ff000
  ```

> 不分配大内存给 Elasticsearch，事实上 JVM 在内存小于32G的时候会采用一个内存对象指针压缩技术。
>
> 在 JAVA 中，所有的对象都分配在堆上，然后有一个指针引用它。指向这些对象的指针大小通常是CPU的字长的大小，不是32bit就是64bit，这取决于你的处理器，指针指向了你的值的精确位置。
>
> 对于32位系统，你的内存最大可使用4G。对于64系统可以使用更大的内存。但是64位的指针意味着更大的浪费，因为你的指针本身大了。浪费内存不算，更糟糕的是，更大的指针在主内存和缓存器（例如LLC, L1等）之间移动数据的时候，会占用更多的带宽。
>
> JAVA 使用一个叫内存指针压缩的技术来解决这个问题。它的指针不再表示对象在内存中的精确位置，而是表示偏移量。这意味着32位的指针可以引用40亿个对象，而不是40亿个字节。最终，也就是说堆内存长到32G的物理内存，也可以用32bit的指针表示。
>
> 一旦你越过那个神奇的30-32G的边界，指针就会切回普通对象的指针，每个对象的指针都变长了，就会使用更多的CPU内存带宽，也就是说你实际上失去了更多的内存。事实上当内存到达40-50GB的时候，有效内存才相当于使用内存对象指针压缩技术时候的32G内存。
>
> 这段描述的意思就是说：即便你有足够的内存，也尽量不要超过32G，因为它浪费了内存，降低了CPU的性能，还要让GC应对大内存。

以上内容摘录自网络 https://my.oschina.net/TOW/blog/598702 

## 禁用内存交换

大多数操作系统都会尽可能的使用更多的内存来缓存文件系统，并饥渴的交换出无用的应用程序内存到磁盘。这会导致部分 JVM 堆空间被交换出内存到磁盘。

内存交换是性能坟墓，会影响节点稳定性，应该避免内存交换。它会造成垃圾收集从毫秒级别延长成分钟级别，会拖慢节点响应甚至可能会影响节点从集群断连。

**启用 `bootstrap.memory_lock`**

第一个选择是在 Linux 系统上使用 mlockall 尝试锁定内存进程地址空间，保护内存被交换出空间。能够实现这一过程的就是在`/etc/elasticsearch/elasticsearch.yml`设置

```yaml
bootstrap.memory_lock: true
```

> mlockall 在尝试分配但无法分配更多可用内存时，可能会导致进程退出。

启动 Elasticsearch 后，通过一下方式检查 mlockall 是否生效了：

```http
GET _nodes?filter_path=**.mlockall
```

如果 mlockall 为 false，则说明 mlockall 请求失败了。需要尝试以下步骤：即[配置系统参数](#配置系统参数) 开端设置的方式。

**禁用所有交换文件**

第二个选择是完全禁用内存交换。通常一台服务器只运行 Elasticsearch 这一个服务，它的内存使用受 JVM 参数控制。这时就可以完全禁止内存交换。

在 Linux 系统上你可以通过命令行`sudo swapoff -a`临时的禁用内存交换。要永久禁止内存交换，需删除`/etc/fstab`文件中任何存在`swap`字样的地方。

**设置 swappiness**

采取第二个选择时，请确保`sysctl`设置内核参数`vm.swappiness=1`，降低swappiness 的值，这个值决定操作系统交换内存的频率。这可以预防正常情况下发生交换。但仍允许os在紧急情况下发生交换。

> swappiness设置为1比设置为0要好，因为在一些内核版本，swappness=0会引发OOM（内存溢出）

## 配置文件描述符

没什么好说的，参见[配置系统参数](#配置系统参数)

检查节点状态方法 （max_file_descriptors）：

```http
GET _nodes/stats/process?filter_path=**.max_file_descriptors
```

## 配置虚拟内存

Elasticsearch 使用 [`hybrid mmapfs / niofs`](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-store.html#default_fs) 存储索引。默认系统给的map count 太少了。

```bash
sysctl -w vm.max_map_count=262144
```

永久生效请修改 `/etc/sysctl.conf` 文件并重启系统。

## 配置线程数

没什么好说的，最少需要 2048 线程。