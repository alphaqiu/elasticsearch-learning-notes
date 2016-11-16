# 安装 x-pack（可选安装）

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

## 关闭 x-pack 中的功能

设置 `/etc/elasticsearch.yml`

```yaml
# ------------------------ xpack ----------------------------
xpack.security.enabled: false
xpack.monitoring.enabled: false
xpack.graph.enabled: false
xpack.watcher.enabled: false
```

