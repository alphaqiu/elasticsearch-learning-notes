# 安装设置 Elasticsearch

## 安装

### 环境需求

JDK 1.8+

假定服务器为 CentOS 7+

假定你懂一些 Linux 系统操作命令

### 安装 elasticsearch

具体可以参考官网，这里仅介绍通过rpm安装方法。

下载安装签名公钥：

```bash
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```

签名指纹：

```
4609 5ACC 8548 582C 1A26 99A9 D27D 666C D88E 42B4
```

#### 从 rpm 仓库安装

在 RedHat 基本发行版系统的 /etc/yum.repos.d/ 目录下创建 elasticsearch.repo 文件，内容包含：

```
[elasticsearch-5.x]
name=Elasticsearch repository for 5.x packages
baseurl=https://artifacts.elastic.co/packages/5.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
```

```bash
sudo yum install elasticsearch
```

查看当前可用版本

```bash
yum list --showduplicates elasticsearch
```

#### 手动下载 rpm 包

请参见[官网](https://www.elastic.co/guide/en/elasticsearch/reference/current/rpm.html#install-rpm)

通过 yum 安装的 Elasticsearch 会创建 同名的用户和群组：elasticsearch。Elasticsearch ==不能运行在root账户下==。

#### SysV init 和 systemd

Elasticsearch 安装好以后不会自动启动，如何启动和停止它取决于系统使用了哪种 SysV .查看方法

```bash
ps -p 1
```

具体设置参见[官网](https://www.elastic.co/guide/en/elasticsearch/reference/current/rpm.html#_sysv_literal_init_literal_vs_literal_systemd_literal_2)

单独启动执行：

```bash
sudo -u elasticsearch /usr/share/elasticsearch/bin/elasticsearch
```

通过服务命令执行：

```bash
# assumed that elasticsearch running on the CentOS 7
systemctl start elasticsearch.service
```

### 安装 x-pack（可选安装）

从ES栈5.0正式版开始，集成 x-pack —— 一种集成了 安全、告警、监控、报告和拓扑能力的插件。它是一场革命性功能，用来替代单独安装 Shield, Watcher, Marvel, Reporting 和 Graph 插件。

[网站链接](https://www.elastic.co/blog/x-pack-5-0-0-released)

安装方法：

```bash
# 通过 yum 安装的 elasticsearch 运行路径在 /usr/share/elasticsearch/bin
bin/elasticsearch-plugin install x-pack
# 通过 yum 安装的 kibanba 运行路径在 /usr/share/kibanba/bin
bin/kibana-plugin install x-pack
```

安装好插件后，它自动创建两个账户 `elastic` 和 `kibana`，密码为 `changeme`

安装好以后默认开启了基本认证。**如果不安装 x-pack 则不会有访问权限的问题存在**。

普通访问

```bash
curl -s "http://localhost:9200"|jq
```

```json
{
  "error": {
    "root_cause": [
      {
        "type": "security_exception",
        "reason": "missing authentication token for REST request [/]",
        "header": {
          "WWW-Authenticate": "Basic realm=\"security\" charset=\"UTF-8\""
        }
      }
    ],
    "type": "security_exception",
    "reason": "missing authentication token for REST request [/]",
    "header": {
      "WWW-Authenticate": "Basic realm=\"security\" charset=\"UTF-8\""
    }
  },
  "status": 401
}
```

需要加入权限

```bash
curl -s --user elastic:changeme "http://localhost:9200"|jq
```

```json
{
  "name": "4vFr63A",
  "cluster_name": "elasticsearch",
  "cluster_uuid": "kT3iHw6wQ4OEw5p8wkRQUw",
  "version": {
    "number": "5.0.0",
    "build_hash": "253032b",
    "build_date": "2016-10-26T04:37:51.531Z",
    "build_snapshot": false,
    "lucene_version": "6.2.0"
  },
  "tagline": "You Know, for Search"
}
```

#### 关闭 x-pack 中的功能

设置 `/etc/elasticsearch.yml`

```yaml
# --------------------------------- xpack ----------------------------------
xpack.security.enabled: false
xpack.monitoring.enabled: false
xpack.graph.enabled: false
xpack.watcher.enabled: false
```



## 设置

### Elasticsearch 配置文件位置

Elasticsearch 有两个配置：

- `elasticsearch.yml` 
- `log4j2.properties`

文件放在 `/etc/elasticsearch` 目录下。yaml 格式



RPM 系统级设置文件 `/etc/sysconfig/elasticsearch` 可在内部设置环境变量。

### 重要的 Elasticsearch 设置项

#### path.data and path.logs

```yaml
path.data: 
	- /path/to/data1
	- /path/to/data2
path.logs: /path/to/log
```

#### cluster.name

服务器节点只能加入到同一个集群名称的集群内。默认集群名称是elasticsearch，请确认修改该名称，防止使用默认集群名称造成错误的节点加入其中。

```yaml
cluster.name: dzhyun-log-collect
```

#### node.name

某个集群节点名称，如果不填，Elasticsearch 将用uuid产生并取前7个字符作为节点名。填写的好处是能够直观了解是哪个节点。

```yaml
node.name: es-node-1
```

或者用系统环境变量：

```yaml
node.name: ${HOSTNAME}
```

#### bootstrap.memory_lock

这是一个极其重要的设置项关乎于你的节点健康状况，将不让 JVM 发生内存数据交换出磁盘。

```yaml
bootstrap.memory_lock: true
```

要使之生效，首先需要更改服务器系统设定。

#### network.host

如果不设置该选项，默认使用本地回环地址（127.0.0.1 和 [::1]）。

为了跟其他节点通信和形成集群，节点需要有一个具体的IP地址。

```yaml
network.host: 10.15.144.105
```

进阶设置说明，可参考[官网 network.host](https://www.elastic.co/guide/en/elasticsearch/reference/current/important-settings.html#network.host)。

#### discovery.zen.ping.unicast.hosts

无需任何网络设置即可开箱即用。Elasticsearch 会绑定本地回环地址并扫描 9300 - 9305端口，尝试连接同一台服务器上的其他节点。这将提供自动建立集群的体验而无需任何设置。

当集群需要分布在不同服务器上时，就需要提供服务器地址列表。

```yaml
discovery.zen.ping.unicast.hosts: 
	- 192.168.0.105:9300
	- 192.168.0.106
	- seeds.domain.com
```

默认端口 9300 可不填。可以是域名

#### discovery.zen.minimum_master_nodes

为避免数据丢失，设置 `discovery.zen.minimum_master_nodes` 也至关重要。

master 候选节点的法定个数 （master候选节点个数/2) + 1。

如果设置3台集群，最少需要 3/2+1 ＝ 2台主节点

```yaml
discovery.zen.minimum_master_nodes: 2
```

由于 Elasticsearch 是动态的，你可以很容易的添加和删除节点，这会改变这个法定个数，如果你不得不修改索引的节点的配置并且重启你的整个集群为了让配置生效，这将是非常痛苦的一件事情。
基于这个原因，minimum_master_nodes （还有一些其它配置），允许通过API调用的方式动态进行配置，当你的集群在线运行的时候，你可以这样修改配置：

```http
PUT /_cluster/settings
{
	“persistent” : {
		“discovery.zen.minimum_master_nodes” : 2
	}
}
```

### Elasticsearch 启动时检查项

#### 检查 Java 堆大小

初始堆大小 ＝ 最大堆大小

`/etc/elasticsearch/jvm.options`

`-Xms16g -Xmx16g`

#### 检查文件描述符

Elasticsearch 会打开非常多的文件句柄，例如 每个分片上的segments，以及其他文件。

临时更改系统设置： `ulimit -n 65536`

永久更改系统设置：`/etc/security/limits.conf`

#### **检查内存锁定**

JVM 的垃圾收集器会访问堆栈上的每一页。如果任何一页被交换出了内存到磁盘，它们必须被重新交换到内存中。Elasticsearch 频繁的服务请求会造成大量的磁盘垃圾碎片。有若干中方法配置系统来禁止磁盘交换。其中一个办法是通过`mlockall`锁定 JVM 堆栈内存。这是通过设置 Elasticsearch 的配置 [bootstrap.memory_lock](#bootstrap.memory_lock)。但是要使 mlockall 生效，需要设置服务器参数。

设置系统内存限制的几种方式：

- 临时设置 `ulimit -l unlimited`


- 永久设置 设置 `/etc/security/limits.conf` memlock unlimited
- systemd 要设置 `LimitMEMLOCK=infinity`

#### **检查最大线程数**

Elasticsearch 会将执行的请求分拆成不同的阶段，并将这些阶段转换到不同的线程池去执行。

Elasticsearch 需要使用至少 2048 线程数。

`/etc/security/limit.conf`文件中的`nproc`设置成至少该数值。

#### **检查最大虚拟内存**

Elasticsearch 和 Lucene 使用 mmap 管理索引地址空间，mmap 脱离 JVM 堆栈并保存着具体的索引数据，但是在内存中又可以很快被访问到。需要无限制的虚拟内存。

`/etc/security/limit.conf`文件中的 `as`设置成`unlimited`

#### **检查最大map count**

接上一条提到的，Elasticsearch 也需要创建大量的内存映射区域的能力。该项检查将检查linux 内核是否允许处理至少262144内存映射区域（只针对linux）。

`sysctl vm.max_map_count=262144` 至少设置 262144

#### 检查 Client JVM

这个jvm.options 里面已经设置好了， 服务器模式。

#### 检查是否使用了序列化垃圾收集器

检查垃圾收集器。序列化垃圾收集器仅适用于单核的CPU以及极小堆。使用它会影响性能。这个在 jvm.options 内也设置好了。要使用 CMS Collector

#### 检查 OnError 和 OnOutOfMemoryError

JVM 设置项 允许设置 OnError 和 OnOutOfMemoryError 过滤器，去触发任意脚本。

但是 Elasticsearch 系统会调用 seccomp 过滤器且，该过滤器禁止 forking

> Seccomp(secure computing)是Linux kernel （自从2.6.23版本之后）所支持的一种简洁的sandboxing机制。它能使一个进程进入到一种“安全”运行模式，该模式下的进程只能调用4种系统调用（system calls），即read(), write(), exit()和sigreturn()，否则进程便会被终止。

因此 OnError 和 OnOutOfMemoryError 过滤器与之不兼容。

instead, upgrade to Java 8u92 and use the JVM flag `ExitOnOutOfMemoryError`. While this does not have the full capabilities of `OnError` nor `OnOutOfMemoryError`, arbitrary forking will not be supported with seccomp enabled.

### 重要的系统设置项

理想情况下，一台服务器部署一套 Elasticserarch 并使用服务器全部可用资源。所以，需要设置服务器配置。

下列给出的清单是必须在生产环境中要处理的：

- 设置 JVM 堆大小
- 禁止 swap 交换
- 提高文件句柄数
- 确保充足的虚拟内存
- 确保充足的线程数

默认情况下，Elasticsearch 假定当前是工作在开发模式，如果以上清单任何一项设置没有配置正确，它会打印警告级别的日志，但仍然可以启动它。

一旦设置好 Elasticsearch 配置文件中的网络相关设置，诸如`network.host`，它会假定你将要转换到生产环境，并且以上提到的警告将提升为Exceptions。这些错误将阻止启动。

#### 配置系统参数

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

#### 配置 JVM 堆大小

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

#### 禁用内存交换

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

#### 配置文件描述符

没什么好说的，参见[配置系统参数](#配置系统参数)

检查节点状态方法 （max_file_descriptors）：

```http
GET _nodes/stats/process?filter_path=**.max_file_descriptors
```

#### 配置虚拟内存

Elasticsearch 使用 [`hybrid mmapfs / niofs`](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-store.html#default_fs) 存储索引。默认系统给的map count 太少了。

```bash
sysctl -w vm.max_map_count=262144
```

永久生效请修改 `/etc/sysctl.conf` 文件并重启系统。

#### 配置线程数

没什么好说的，最少需要 2048 线程。