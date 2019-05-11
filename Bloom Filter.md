# Bloom Filter
布隆过滤器(bf)，相比set，更节省空间，但存在一定的预判率。  
给定一个值，如果bf说不存在，那就一定不存在；  
如果bf说存在，大概率是存在的。
## 编译
```
shell> mkdir BloomFilter
shell> cd BloomFilter
shell> git clone https://github.com/RedisLabsModules/redisbloom.git .
shell> make
```

## 加载
```
shell> redis-server --loadmodule ./rebloom.so &
```

## 使用
- bf.reserve
syntax: bf.reserve key error_rate initial_size  
显式创建一个布隆过滤器，同时指定初始容量和误判率。
```
redis-cli> bf.reserve id 0.001 1000 
OK
```

- bf.add
syntax: bf.add key value  
添加一个元素
```
redis-cli> bf.add id 1
(integer) 1
redis-cli> bf.add id 2
(integer) 1
```


- bf.exists
syntax: bf.exists key value  
判断 key对应的bloom filer是否已有这个元素
```
redis-cli> bf.exists id 1
(integer) 1
redis-cli> bf.exists id 3
(integer) 0
```