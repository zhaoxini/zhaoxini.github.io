---
title: Jedis线程回收问题
date: 2021-01-10 22:07:48
tags: Jedis
---

> 问题：redis.clients.jedis.exceptions.JedisConnectionException:Could not get a resource from the pool



查询问题原因，猜测以下几点

1. Redis没启动
2. 资源池参数不合理：比如QPS高，连接池小。
3. 因为用的Jedis, jedis线程池用完了没有归还



进入redis客户端



```shell
> info client
# Client
connected_clients:628
client_longest_output_list:0
client_biggest_input_buf:0
blocked_clients:0
```

查过几次连接数居高不变

config get maxclients

查询出来也没问题





然后继续查询Jedis回收问题

网上多数都说没有finnally回收

但是项目代码是有的



然后开始Debug，发现确实没有走到那步。

然后找原因发现

```java
....
}finally {
  if (jedis != null) {
    jedis.close();
  }
}
```

。。。。

这不应该是jedis.close啊，应该是jedisPool.close()啊，粗心了。









正好在这总结下jedis.close 和 jedisPool.close的区别

在jedis3.0后，jedisPool.returnResource()方法是被弃用的

....未完待续

