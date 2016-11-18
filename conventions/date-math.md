```
<static_name{date_math_expr{date_format|time_zone}}>
```

**static_name**

静态名称，前缀

**date_math_expr**

日期公式表达式

**date_format**

日期格式化字符串格式，默认YYYY.MM.dd

**time_zone**

时区 默认 utc

<logstash-{now/d}>  logstash-2016.11.17  当前日期，精确到某一日

<logstash-{now/M}> logstash-2016.11.01  当前日期，精确到月的第一天

<logstash-{now/M{YYYY-MM}}> logstash-2016-11  当前日期，格式化

<logstash-{now/M-1M}>  logstash-2016.10.01