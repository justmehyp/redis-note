# keys & scan

## keys
想要查看当前redis中有哪些 key 时，可以使用 keys 命令  
syntax: keys pattern  
pattern 支持通配符  

先造一点数据：
```
redis-cli> set lock:a ""
OK
redis-cli> set lock:b ""
OK
redis-cli> set lock:cc ""
OK
redis-cli> lpush mq:msg hello
(integer) 1
redis-cli> lpush mq:msg world
(integer) 2
redis-cli> hset user:1 name xiaoming
(integer) 1
redis-cli> hset user:1 age 1
(integer) 1
redis-cli> sadd id 0 1 2
(integer) 3
redis-cli> zadd student_score 60 xiaoming
(integer) 1
redis-cli> zadd student_score 90 xiaohong
(integer) 1
redis-cli> zadd student_score 59 xiaowang
(integer) 1
```

示例：  
1）获取所有 key:
```
redis-cli> keys *
1) "student_score"
2) "mq:msg"
3) "lock:cc"
4) "id"
5) "lock:b"
6) "lock:a"
7) "user:1"
```

2）？通配一个字符
```
redis-cli> keys lock:?
1) "lock:b"
2) "lock:a"
```
可以看到，lock:cc 没有筛选出来。

3) *匹配模糊匹配多个字符
```
redis-cli> keys lock:*
1) "lock:cc"
2) "lock:b"
3) "lock:a"
redis-cli> keys *c*
1) "lock:cc"
2) "student_score"
3) "lock:b"
4) "lock:a"
```

## scan
keys 命令会扫描所有的 key，如果 key 很多的话，由于 redis 是单线程的，会导致扫描时间过长，影响其他客户端的操作，导致卡顿。所以，生产环境应该慎用。  
scan 命令支持游标遍历，还可以指定 count 查询数量，这样可以每次小批量的遍历，避免长时间独占线程。  
syntax: scan cursor [MATCH pattern] [COUNT count]  
第一次查询时，cursor = 0，命令返回时，第一个元素时下一次查询的 cursor。  
当返回的 cursor = 0 时，遍历结束；  
当返回的 cursor != 0，遍历未结束，即使结果为 empty list

示例：  
1）模糊匹配，不带 count 参数，效果等同于 keys 的。
```
redis-cli> scan 0 
1) "0"
2) 1) "lock:cc"
   2) "student_score"
   3) "mq:msg"
   4) "id"
   5) "lock:b"
   6) "lock:a"
   7) "user:1"
redis-cli> scan 0  match i?
1) "0"
2) 1) "id"
redis-cli> scan 0 match *c*
1) "0"
2) 1) "lock:cc"
   2) "student_score"
   3) "lock:b"
   4) "lock:a"
```

2）带 count，小批量查询 key
```
redis-cli> scan 0 match * count 2
1) "4"
2) 1) "lock:cc"
   2) "student_score"
redis-cli> scan 4 match * count 2
1) "3"
2) 1) "mq:msg"
   2) "id"
   3) "lock:b"
redis-cli> scan 3 match * count 2
1) "0"
2) 1) "lock:a"
   2) "user:1"
```

## 更多的 scan 命令
redis 中所有的 key 都是放在一个 map 当中的，因此，类似的，其 hash 结果也应该有 scan 命令；  
set 底层是用 hash 实现的，应该也有 scan；  
zset 亦是如此。
hash 对应的 scan 命令就是： hscan  
set 的 scan 命令： sscan  
zset 的 scan 命令： zscan  
这些命令的参数和 scan 一摸一样。

## 大 key 扫描
官方提供了一个命令参数： ```redis-cli --bigkeys``` 。 一次扫描时间可能很长，可以加多一个参数：```redis-cli --bigkeys -i 0.1```,  这样，每 100 条 scan 命令后，会休眠 0.1 s。 