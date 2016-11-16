# 安装运行 Elasticsearch

具体可以参考官网，这里仅介绍通过rpm安装方法。

下载安装签名公钥：

```bash
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```

签名指纹：

```
4609 5ACC 8548 582C 1A26 99A9 D27D 666C D88E 42B4
```

## 从 rpm 仓库安装

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

## 手动下载 rpm 包

请参见[官网](https://www.elastic.co/guide/en/elasticsearch/reference/current/rpm.html#install-rpm)

通过 yum 安装的 Elasticsearch 会创建 同名的用户和群组：elasticsearch。

Elasticsearch **不能运行在root账户下**。

## SysV init 和 systemd

Elasticsearch 安装好以后不会自动启动，如何启动和停止它取决于系统使用了哪种 SysV .查看方法

```bash
ps -p 1
```

具体设置参见[官网](https://www.elastic.co/guide/en/elasticsearch/reference/current/rpm.html#_sysv_literal_init_literal_vs_literal_systemd_literal_2)

单独启动执行：

```bash
sudo -u elasticsearch /usr/share/elasticsearch/bin/elasticsearch
```

如果想要在后台以守护进程模式运行，添加 `-d` 参数。

通过服务命令执行：

```bash
# assumed that elasticsearch running on the CentOS 7
systemctl start elasticsearch.service
```

打开另一终端测试：

```bash
curl "http://localhost:9200/?pretty"
```

你能看到以下返回信息：

```json
{
   "status": 200,
   "name": "Shrunken Bones",
   "version": {
      "number": "5.0.0",
      "lucene_version": "4.10"
   },
   "tagline": "You Know, for Search"
}
```

Elasticsearch 开箱即用，无需任何多余的设置。

## 集群和节点

**节点(node)**是一个运行着的Elasticsearch实例。

**集群(cluster)**是一组具有相同`cluster.name`的节点集合，他们协同工作，共享数据并提供故障转移和扩展功能，当然一个节点也可以组成一个集群。

你最好找一个合适的名字来替代`cluster.name`的默认值，比如你自己的名字，这样可以防止一个新启动的节点加入到相同网络中的另一个同名的集群中。

你可以通过修改`config/`目录下的`elasticsearch.yml`文件，然后重启ELasticsearch来做到这一点。当Elasticsearch在前台运行，可以使用`Ctrl-C`快捷键终止，或者你可以调用`shutdown` API来关闭：

```bash
curl -XPOST 'http://localhost:9200/_shutdown'
```

