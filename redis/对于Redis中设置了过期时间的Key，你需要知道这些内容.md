# 对于Redis中设置了过期时间的Key，你需要知道这些内容

上一篇文章我们讲到了Redis的内存淘汰策略（[传送门](https://juejin.im/post/5d674ac2e51d4557ca7fdd70)），这次跟我一起看一下Redis的过期策略。

熟悉Redis的同学应该知道，Redis的每个Key都可以设置一个过期时间，当达到过期时间的时候，这个key就会被自动删除。

### 在为key设置过期时间需要注意的事项

#### 1、 [DEL](http://doc.redisfans.com/key/del.html)/[SET](http://doc.redisfans.com/string/set.html)/[GETSET](http://doc.redisfans.com/string/getset.html)等命令会清除过期时间

在使用**DEL、SET、GETSET**等会覆盖key对应value的命令操作一个设置了过期时间的key的时候，会导致对应的key的过期时间被清除。

```shell
//设置mykey的过期时间为300s
127.0.0.1:6379> set mykey hello ex 300
OK
//查看过期时间
127.0.0.1:6379> ttl mykey
(integer) 294
//使用set命令覆盖mykey的内容
127.0.0.1:6379> set mykey olleh
OK
//过期时间被清除
127.0.0.1:6379> ttl mykey
(integer) -1
复制代码
```

#### 2、[INCR](http://doc.redisfans.com/string/incr.html)/[LPUSH](http://doc.redisfans.com/list/lpush.html)/[HSET](http://doc.redisfans.com/hash/hset.html)等命令则不会清除过期时间

而在使用**INCR/LPUSH/HSET**这种只是修改一个key的value，而不是覆盖整个value的命令，则不会清除key的过期时间。 INCR：

```shell
//设置incr_key的过期时间为300s
127.0.0.1:6379> set incr_key 1 ex 300
OK
127.0.0.1:6379> ttl incr_key
(integer) 291
//进行自增操作
127.0.0.1:6379> incr incr_key
(integer) 2
127.0.0.1:6379> get incr_key
"2"
//查询过期时间，发现过期时间没有被清除
127.0.0.1:6379> ttl incr_key
(integer) 277
复制代码
```

LPUSH：

```shell
//新增一个list类型的key，并添加一个为1的值
127.0.0.1:6379> LPUSH list 1
(integer) 1
//为list设置300s的过期时间
127.0.0.1:6379> expire list 300
(integer) 1
//查看过期时间
127.0.0.1:6379> ttl list
(integer) 292
//往list里面添加值2
127.0.0.1:6379> lpush list 2
(integer) 2
//查看list的所有值
127.0.0.1:6379> lrange list 0 1
1) "2"
2) "1"
//能看到往list里面添加值并没有使过期时间清除
127.0.0.1:6379> ttl list
(integer) 252
复制代码
```

#### 3、[PERSIST](http://doc.redisfans.com/key/persist.html)命令会清除过期时间

当使用**PERSIST**命令将一个设置了过期时间的key转变成一个持久化的key的时候，也会清除过期时间。

```shell
127.0.0.1:6379> set persist_key haha ex 300
OK
127.0.0.1:6379> ttl persist_key
(integer) 296
//将key变为持久化的
127.0.0.1:6379> persist persist_key
(integer) 1
//过期时间被清除
127.0.0.1:6379> ttl persist_key
(integer) -1
复制代码
```

#### 4、使用[RENAME](http://doc.redisfans.com/key/rename.html)命令，老key的过期时间将会转到新key上

在使用例如：**RENAME KEY_A KEY_B**命令将KEY_A重命名为KEY_B，不管KEY_B有没有设置过期时间，新的key KEY_B将会继承KEY_A的所有特性。

```shell
//设置key_a的过期时间为300s
127.0.0.1:6379> set key_a value_a ex 300
OK
//设置key_b的过期时间为600s
127.0.0.1:6379> set key_b value_b ex 600
OK
127.0.0.1:6379> ttl key_a
(integer) 279
127.0.0.1:6379> ttl key_b
(integer) 591
//将key_a重命名为key_b
127.0.0.1:6379> rename key_a key_b
OK
//新的key_b继承了key_a的过期时间
127.0.0.1:6379> ttl key_b
(integer) 248
复制代码
```

> 这里篇幅有限，我就不一一将key_a重命名到key_b的各个情况列出来，大家可以在自己电脑上试一下key_a设置了过期时间，key_b没设置过期时间这种情况。

#### 5、使用[EXPIRE](http://doc.redisfans.com/key/expire.html)/[PEXPIRE](http://doc.redisfans.com/key/pexpire.html)设置的过期时间为负数或者使用[EXPIREAT](http://doc.redisfans.com/key/expireat.html)/[PEXPIREAT](http://doc.redisfans.com/key/pexpireat.html)设置过期时间戳为过去的时间会导致key被删除

EXPIRE：

```shell
127.0.0.1:6379> set key_1 value_1
OK
127.0.0.1:6379> get key_1
"value_1"
//设置过期时间为-1
127.0.0.1:6379> expire key_1 -1
(integer) 1
//发现key被删除
127.0.0.1:6379> get key_1
(nil)
复制代码
```

EXPIREAT：

```shell
127.0.0.1:6379> set key_2 value_2
OK
127.0.0.1:6379> get key_2
"value_2"
//设置的时间戳为过去的时间
127.0.0.1:6379> expireat key_2 10000
(integer) 1
//key被删除
127.0.0.1:6379> get key_2
(nil)
复制代码
```

#### 6、[EXPIRE](http://doc.redisfans.com/key/expire.html)命令可以更新过期时间

对一个已经设置了过期时间的key使用expire命令，可以更新其过期时间。

```shell
//设置key_1的过期时间为100s
127.0.0.1:6379> set key_1 value_1 ex 100
OK
127.0.0.1:6379> ttl key_1
(integer) 95
//更新key_1的过期时间为300s
127.0.0.1:6379> expire key_1 300
(integer) 1
127.0.0.1:6379> ttl key_1
(integer) 295
复制代码
```

> 在Redis2.1.3以下的版本中，使用expire命令更新一个已经设置了过期时间的key的过期时间会失败。并且对一个设置了过期时间的key使用LPUSH/HSET等命令修改其value的时候，会导致Redis删除该key。

### Redis的过期策略

那你有没有想过一个问题，Redis里面如果有大量的key，怎样才能高效的找出过期的key并将其删除呢，难道是遍历每一个key吗？假如同一时期过期的key非常多，Redis会不会因为一直处理过期事件，而导致读写指令的卡顿。

> 这里说明一下，Redis是单线程的，所以一些耗时的操作会导致Redis卡顿，比如当Redis数据量特别大的时候，使用keys * 命令列出所有的key。

实际上Redis使用**懒惰删除+定期删除**相结合的方式处理过期的key。

#### 懒惰删除

所谓**懒惰删除**就是在客户端访问该key的时候，redis会对key的过期时间进行检查，如果过期了就立即删除。

这种方式看似很完美，在访问的时候检查key的过期时间，不会占用太多的额外CPU资源。但是如果一个key已经过期了，如果长时间没有被访问，那么这个key就会一直存留在内存之中，严重消耗了内存资源。

#### 定期删除

**定期删除**的原理是，Redis会将所有设置了过期时间的key放入一个字典中，然后每隔一段时间从字典中随机一些key检查过期时间并删除已过期的key。

Redis默认每秒进行10次过期扫描：

1. **从过期字典中随机20个key**
2. **删除这20个key中已过期的**
3. **如果超过25%的key过期，则重复第一步**

同时，为了保证不出现循环过度的情况，Redis还设置了扫描的时间上限，默认不会超过25ms。

### 参考资料

[redis.io/commands/ex…](https://redis.io/commands/expire#expire-accuracy)


作者：千山qianshan链接：https://juejin.im/post/5d6bda096fb9a06acc009dc8来源：掘金著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。