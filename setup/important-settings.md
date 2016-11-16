# 重要的 Elasticsearch 设置项

## path.data and path.logs

```yaml
path.data: 
	- /path/to/data1
	- /path/to/data2
path.logs: /path/to/log
```

## cluster.name

服务器节点只能加入到同一个集群名称的集群内。默认集群名称是elasticsearch，请确认修改该名称，防止使用默认集群名称造成错误的节点加入其中。

```yaml
cluster.name: dzhyun-log-collect
```

## node.name

某个集群节点名称，如果不填，Elasticsearch 将用uuid产生并取前7个字符作为节点名。填写的好处是能够直观了解是哪个节点。

```yaml
node.name: es-node-1
```

或者用系统环境变量：

```yaml
node.name: ${HOSTNAME}
```

## bootstrap.memory_lock

这是一个极其重要的设置项关乎于你的节点健康状况，将不让 JVM 发生内存数据交换出磁盘。

```yaml
bootstrap.memory_lock: true
```

要使之生效，首先需要更改服务器系统设定。

## network.host

如果不设置该选项，默认使用本地回环地址（127.0.0.1 和 [::1]）。

为了跟其他节点通信和形成集群，节点需要有一个具体的IP地址。

```yaml
network.host: 10.15.144.105
```

进阶设置说明，可参考[官网 network.host](https://www.elastic.co/guide/en/elasticsearch/reference/current/important-settings.html#network.host)。

## discovery.zen.ping.unicast.hosts

无需任何网络设置即可开箱即用。Elasticsearch 会绑定本地回环地址并扫描 9300 - 9305端口，尝试连接同一台服务器上的其他节点。这将提供自动建立集群的体验而无需任何设置。

当集群需要分布在不同服务器上时，就需要提供服务器地址列表。

```yaml
discovery.zen.ping.unicast.hosts: 
	- 192.168.0.105:9300
	- 192.168.0.106
	- seeds.domain.com
```

默认端口 9300 可不填。可以是域名

## discovery.zen.minimum_master_nodes

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

### 