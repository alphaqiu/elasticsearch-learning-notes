```http
PUT _cluster/settings
{
  "transient": {
    "discovery.zen.minimum_master_nodes": 2
  }
}
```

顺序 

创建索引 -> 建立映射关系 -> 索引数据

**关闭自动创建索引**

如果要关闭自动创建索引，需要在所有节点上都这样操作，设置 elasticsearch.yml 配置文件

```yaml
action.auto_create_index: false
```

自动创建索引可以配置白名单或黑名单：

```yaml
action.auto_create_index: +logstash*,-dzhyun*
```

加号代表白名单，减号代表黑名单

**关闭自动映射**

索引设置

```json
{
  "index.mapper.dynamic": false
}
```

index.write.wait_for_active_shards

To alter this behavior per operation, the `wait_for_active_shards` request parameter can be used



realtime=false



node.max_local_storage_nodes



```
* - nofile 65536
* - memlock unlimited
* - as unlimited
* - nproc unlimited
```

