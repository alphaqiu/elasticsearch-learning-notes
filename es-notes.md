# Elasticsearch 5.0 笔记

## 安装设置 Elasticsearch

### 安装

#### 安装 elasticsearch

具体可以参考官网，这里仅介绍通过rpm安装方法。

下载安装签名公钥：

```bash
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```

签名指纹：

```
4609 5ACC 8548 582C 1A26 99A9 D27D 666C D88E 42B4
```

##### 从 rpm 仓库安装

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

##### 手动下载 rpm 包

请参见[官网](https://www.elastic.co/guide/en/elasticsearch/reference/current/rpm.html#install-rpm)

##### SysV init 和 systemd

Elasticsearch 安装好以后不会自动启动，如何启动和停止它取决于系统使用了哪种 SysV .查看方法

```bash
ps -p 1
```

具体设置参见[官网](https://www.elastic.co/guide/en/elasticsearch/reference/current/rpm.html#_sysv_literal_init_literal_vs_literal_systemd_literal_2)

#### 安装 x-pack（可选安装）

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

### 设置

#### Elasticsearch 配置文件位置

Elasticsearch 有两个配置：

- `elasticsearch.yml` 
- `log4j2.properties`

文件放在 `/etc/elasticsearch` 目录下。yaml 格式



RPM 系统级设置文件 `/etc/sysconfig/elasticsearch` 可在内部设置环境变量。

#### 重要的 Elasticsearch 设置项

##### path.data and path.logs

```yaml
path.data: 
	- /path/to/data1
	- /path/to/data2
path.logs: /path/to/log
```

##### cluster.name

服务器节点只能加入到同一个集群名称的集群内。默认集群名称是elasticsearch，请确认修改该名称，防止使用默认集群名称造成错误的节点加入其中。

```yaml
cluster.name: dzhyun-log-collect
```

##### node.name

某个集群节点名称，如果不填，Elasticsearch 将用uuid产生并取前7个字符作为节点名。填写的好处是能够直观了解是哪个节点。

```yaml
node.name: es-node-1
```

或者用系统环境变量：

```yaml
node.name: ${HOSTNAME}
```

##### bootstrap.memory_lock

这是一个极其重要的设置项关乎于你的节点健康状况，将不让 JVM 发生内存数据交换出磁盘。

```yaml
bootstrap.memory_lock: true
```

要使之生效，首先需要更改服务器系统设定。

##### network.host

如果不设置该选项，默认使用本地回环地址（127.0.0.1 和 [::1]）。

为了跟其他节点通信和形成集群，节点需要有一个具体的IP地址。

```yaml
network.host: 10.15.144.105
```

进阶设置说明，可参考[官网 network.host](https://www.elastic.co/guide/en/elasticsearch/reference/current/important-settings.html#network.host)。

##### discovery.zen.ping.unicast.hosts

无需任何网络设置即可开箱即用。Elasticsearch 会绑定本地回环地址并扫描 9300 - 9305端口，尝试连接同一台服务器上的其他节点。这将提供自动建立集群的体验而无需任何设置。

当集群需要分布在不同服务器上时，就需要提供服务器地址列表。

```yaml
discovery.zen.ping.unicast.hosts: 
	- 192.168.0.105:9300
	- 192.168.0.106
	- seeds.domain.com
```

默认端口 9300 可不填。可以是域名

##### discovery.zen.minimum_master_nodes

为避免数据丢失，设置 `discovery.zen.minimum_master_nodes` 也至关重要。

主节点选举最少节点数 （合格的主节点数/2) + 1。如果设置3台集群，最少需要 3/2+1 ＝ 2台主节点

```yaml
discovery.zen.minimum_master_nodes: 2
```

#### Elasticsearch 启动时检查项

##### 检查 Java 堆大小

初始堆大小 ＝ 最大堆大小

`-Xms16g -Xmx16g`

##### 检查文件描述符

Elasticsearch 会打开非常多的文件句柄，例如 每个分片上的segments，以及其他文件。

临时更改系统设置： `ulimit -n 65536`

永久更改系统设置：`/etc/security/limits.conf`

##### **检查内存锁定**

JVM 的垃圾收集器会访问堆栈上的每一页。如果任何一页被交换出了内存到磁盘，它们必须被重新交换到内存中。Elasticsearch 频繁的服务请求会造成大量的磁盘垃圾碎片。有若干中方法配置系统来禁止磁盘交换。其中一个办法是通过`mlockall`锁定 JVM 堆栈内存。这是通过设置 Elasticsearch 的配置 [bootstrap.memory_lock](#bootstrap.memory_lock)。但是要使 mlockall 生效，需要设置服务器参数。

设置系统内存限制的几种方式：

- 临时设置 `ulimit -l unlimited`


- 永久设置 设置 `/etc/security/limits.conf` memlock unlimited
- systemd 要设置 `LimitMEMLOCK=infinity`

##### **检查最大线程数**

Elasticsearch 会将执行的请求分拆成不同的阶段，并将这些阶段转换到不同的线程池去执行。

Elasticsearch 需要使用至少 2048 线程数。

`/etc/security/limit.conf`文件中的`nproc`设置成至少该数值。

##### **检查最大虚拟内存**

Elasticsearch 和 Lucene 使用 mmap 管理索引地址空间，mmap 脱离 JVM 堆栈并保存着具体的索引数据，但是在内存中又可以很快被访问到。需要无限制的虚拟内存。

`/etc/security/limit.conf`文件中的 `as`设置成`unlimited`

##### **检查最大map count**

接上一条提到的，Elasticsearch 也需要创建大量的内存映射区域的能力。该项检查将检查linux 内核是否允许处理至少262144内存映射区域（只针对linux）。

`sysctl vm.max_map_count=262144` 至少设置 262144

##### 检查 Client JVM

这个jvm.options 里面已经设置好了， 服务器模式。

##### 检查是否使用了序列化垃圾收集器

检查垃圾收集器。序列化垃圾收集器仅适用于单核的CPU以及极小堆。使用它会影响性能。这个在 jvm.options 内也设置好了。要使用 CMS Collector

##### 检查 OnError 和 OnOutOfMemoryError

JVM 设置项 允许设置 OnError 和 OnOutOfMemoryError 过滤器，去触发任意脚本。

但是 Elasticsearch 系统会调用 seccomp 过滤器且，该过滤器禁止 forking

> Seccomp(secure computing)是Linux kernel （自从2.6.23版本之后）所支持的一种简洁的sandboxing机制。它能使一个进程进入到一种“安全”运行模式，该模式下的进程只能调用4种系统调用（system calls），即read(), write(), exit()和sigreturn()，否则进程便会被终止。

因此 OnError 和 OnOutOfMemoryError 过滤器与之不兼容。

instead, upgrade to Java 8u92 and use the JVM flag `ExitOnOutOfMemoryError`. While this does not have the full capabilities of `OnError` nor `OnOutOfMemoryError`, arbitrary forking will not be supported with seccomp enabled.

#### 重要的系统设置项

