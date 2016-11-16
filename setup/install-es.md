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

通过 yum 安装的 Elasticsearch 会创建 同名的用户和群组：elasticsearch。Elasticsearch ==不能运行在root账户下==。

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

通过服务命令执行：

```bash
# assumed that elasticsearch running on the CentOS 7
systemctl start elasticsearch.service
```

### 