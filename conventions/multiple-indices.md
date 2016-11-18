# 多索引

大多数涉及到参数`index`的API都支持跨索引搜索，使用 `test1,test2,test3`标记法(或`_all`查询所有索引)，它同样支持通配符，比如：`test*` , `*test` , `te*t`或`*test*`。前导符`+`表示`add`，`-`表示`remove`。例如：`+test*,-test3`。

所有多索引 API 都支持以下查询参数：

**ignore_unavailable**

参数值： true 或 false

控制忽略指定了不可用的索引（不存在、关闭、异常等）

**allow_no_indices**

参数值：true 或 false

控制通配符索引表达式找不到索引时的行为。同样支持 _all 或 *



**expand_wildcards**

参数值：open、 closed、none、all







