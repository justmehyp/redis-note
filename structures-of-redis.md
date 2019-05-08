# Redis 的数据结构及其操作

## string
类似 Java 当中的 ArrayList, string 是 redis 最基本的数据结构，是其他数据结构的基础：多个 string 就是 list, k-v 形式的 hash，其 key 和 value 都是 string，set 和 zset 本质上就是 hash。
### operations
- set key value [EX seconds | PX milliseconds] [NX | XX]  
  value 是一个字符串，如果有空格，整个字符串需要用""括起来  
  [] 内为 optional 的参数, | 代表二选一哦  
  EX, PX, NX, XX 这种大写的，一般都是照抄  
  EX 表示过期时间，单位：秒  
  PX 表示过期时间，单位：毫秒  
  NX 表示 not exists 的意思，当这个 key 不存在的时候，操作才会成功  
  XX 表示 exists 的意思，当这个 key 存在时，操作才会成功  
  
  示例  
  (1) 最简单的用法：
    ```
    如果 key 原先不存在：
    > set name Tom
    OK
    > get name
    "Tom"

    如果 key 原先已经存在，会覆盖旧值：
    > set name Jack
    OK
    > get name
    "Jack"
    ```
    (2) 设置过期时间为 5 秒，过期后 key 会失效：
    ```
    > set name Tom ex 5
    OK

    过期前，可以 get 到 value：
    > get name
    "Tom"

    过期后，get 到 空(nil):
    > get name
    (nil)
    ```
    (3) key 不存在才能 set:
    ```
    > set name Tom nx
    OK

    再次 set 同一个 key:
    > set name Tom nx
    (nil)
    ```
    (4) key 存在才能 set:
    ```
    > set name Tom
    OK
    > set name Tom xx
    OK

    > del name
    (integer) 1
    > set name Tom xx
    (nil)
    ```
    (5) 综合，设置过期时间，并且只有在 key 不存在时，才能操作成功 （这是利用 redis 实现分布式锁的关键）
    ```
    过期时间为 5s：
    > set name Tom ex 5 nx
    OK

    未过期前：
    > set name Tom ex 5 nx
    (nil)

    过期后：
    > set name Tom ex 5 nx
    OK
    ```

- get key

  非常简单，获取 key 对应的 value  
  示例：
  ```
  > set name Tom
  OK
  > get name
  "Tom"
  ```

- mset key value [key value ...]

  Set multiple keys to multiple values  
  批量设置，好处是减少网络传输的次数，redis 客户端和服务端是通过网络连接的  
  示例：
    ```
      > mset name Tom age 20 email Tom@qq.com
      OK
    ```

- mget key [key ...]

  get multiple values of given keys  
  批量获取，好处同样是减少网络传输的次数  
  示例：
    ```
    > mget name age
    1) "Tom"
    2) "20"
    ```

- setnx key value

  set value if key not exists  
  当这个 key 不存在的时候，操作才会成功，相当于: set key value nx，不同的地方是返回值不同  
  示例：
    ```
    - 失败的例子：
    > set name Tom
    OK
    > setnx name Tom
    (integer) 0
    > set name Tom nx
    (nil)

    - 成功的例子：
    > del name
    (integer) 1
    > set name Tom nx
    OK
    > del name
    (integer) 1
    > setnx name Tom
    (integer) 1
    ```

- setex key seconds value  
  set value with expiration  
  设置 key 对应的 value，同时设置 key 的过期时间，相当于：set key value ex seconds  
  示例：
    ```
    > setex name 5 Tom
    OK
    > set name Tom ex 5
    OK
    ```

- strlen key  
  get the length of key's value  
  获取 key 对应的 value 字符串的长度  
  示例：
    ```
    > set name Tom
    OK
    > strlen name
    (integer) 3
    ```
- incr key  
  如果 value 是一个 整型字符串，会自增 1， 类似 Java 的 ++ 操作符  
  示例:
  ```
  > set age 1
  OK
  > incr age
  (integer) 2

  如果是非整型，则会报错：
  > set name Tom
  OK
  > incr name
  (error) ERR value is not an integer or out of range
  > set height 1.80
  OK
  > incr height
  (error) ERR value is not an integer or out of range

  redis 中的整型是一个有符号的 64 bits 的数字，最大值是：Java 中的 Long.MAX_VALUE = 9223372036854775807，
  incr 如果大于这个这个数，就会报错：
  > set num 9223372036854775806
  OK
  > incr num
  (integer) 9223372036854775807
  > incr num
  (error) ERR increment or decrement would overflow
  ```

- decr key  
  如果 value 是一个 整型字符串，会自减 1， 类似 Java 的 -- 操作符  
    示例:
    ```
    > set age 1
    OK
    > decr age
    (integer) 0

    如果是非整型，则会报错：

    redis 中的整型是一个有符号的 64 bits 的数字，最小值是 Java 中的 Long.MIN_VALUE =  -9223372036854775808
    > set num -9223372036854775807
    OK
    > decr num
    (integer) -9223372036854775808
    > decr num
    (error) ERR increment or decrement would overflow
    ```

- incrby key increment  
  如果 value 是一个 整型字符串， 类似 Java 的 += 操作符  
  示例:
    ```
    > set age 1
    OK
    > incrby age 10
    (integer) 11
    ```

- decrby key increment   
  如果 value 是一个 整型字符串， 类似 Java 的 -= 操作符  
  示例:
    ```
    > set age 1
    OK
    > decrby age 10
    (integer) -9
    ```

- incrbyfloat key increment  
  incr by float, 加上一个浮点数  
  示例：
  ```
  > set num .1
  OK
  > incrbyfloat num .1
  "0.2"
  ```

- getset key value  
  set new value and return the old value  
  示例：
  ```
  > getset name Tom
  (nil)
  > getset name Tom
  "Tom"
  > getset name Jack
  "Tom"
  ```

- append key value  
  追加字符串到原来的字符串  
  示例：
  ```
  > set statement hello
  OK
  > append statement " world"
  (integer) 11
  > get statement
  "hello world"
  ```

## list
类似 Java 当中的 LinkedList  
- lpush key value [value ...]  
- rpush key value [value ...]  
  lpush 这里开头的 l 可以理解为: left，从左边往 list 压入一个新的 字符串    
  rpush 这里开头的 r 可以理解为: right，从右边往 list 压入一个新的 字符串  
  示例：
  ```
  (1) lpush
  > del statement
  (integer) 1
  > lpush statement hello world
  (integer) 2
  > lrange statement 0 -1   # 这个语句表示，从左往右，获取 list 的所有 item, 后面介绍
  1) "world"
  2) "hello"

  (2) rpush
  > del statement
  (integer) 1
  > rpush statement hello world
  (integer) 2
  > lrange statement 0 -1
  1) "hello"
  2) "world"
  ```

- lpop key  
- rpop key  
  lpop 这里开头的 l 可以理解为: left，弹出 list 中最左的 item  
  rpop 这里开头的 r 可以理解为: right，弹出 list 中最右的 item  
  示例：  
  (1) 从左边 push, 从左边 pop: 实现一个栈  
  ```
  > del lang
  (integer) 0
  > lpush lang Java
  (integer) 1
  > lpush lang C++ Python
  (integer) 3
  > lpop lang
  "Python"
  > lpop lang
  "C++"
  > lpop lang
  "Java"
  > lpop lang
  (nil)
  ```
  (2) 从右边 push, 从左边 pop: 实现一个队列  
  ```
  > del lang
  (integer) 1
  > rpush lang Java
  (integer) 1
  > rpush lang C++ Python
  (integer) 3
  > lpop lang
  "Java"
  > lpop lang
  "C++"
  > lpop lang
  "Python"
  > lpop
  ```

- llen key  
  获取 list 的 item 数量  
  示例：
  ```
  > del lang
  (integer) 0
  > rpush lang Java C++ C C#
  (integer) 4
  > llen lang
  (integer) 4
  ```

- lindex key index  
  类似 Java 当中的 List#get(int index), 慎用，效率是 O(n), 以后一起看看 redis 的实现原来  
  示例  
  ```
  > del lang
  (integer) 0
  > rpush lang Java C++ C C
  (integer) 4
  > lindex lang 0
  "Java"
  > lindex lang 3
  "C#"
  ```

- lrange key start stop  
  从左往右，获取从 index = start 到 index = stop 的所有元素  
  index 从 0 开始， -1 代表倒数第 1 个元素， -2 代表倒数第 2 个元素  
  示例：
  ```
  > del lang
  (integer) 1
  > rpush lang Java C++ C Python Ruby
  (integer) 5
  > lrange lang 0 0
  1) "Java"
  > lrange lang 0 -1  # 从 0 到 -1（倒数第 1 个），全部
  1) "Java"
  2) "C++"
  3) "C"
  4) "Python"
  5) "Ruby"
  ```

- rpoplpush source destination  
  rpop a item from source, then lpush it to destination  
  示例：
  ```
  > del a b
  (integer) 1
  > lpush a Java C++ python
  (integer) 3
  > rpoplpush a b
  "Java"
  > lrange a 0 -1
  1) "python"
  2) "C++"
  > lrange b 0 -1
  1) "Java"
  > lrange a 0 -1
  1) "python"
  > lrange b 0 -1
  1) "C++"
  2) "Java"
  ```

- blpop key [key ...] timeout  
- brpop key [key ...] timeout  
  b 代表 block 阻塞的意思，当 list 为空时，客户端会阻塞  
  timeout 是超时时间，单位：秒，超时后自动返回  
  示例1， 超时失败的情况：
  ```
    > del a
    (integer) 0
    127.0.0.1:6379> blpop a 2  # 等待 2 秒
    (nil)
    (2.06s)
  ```

  示例2，等待成功  
  需要开 2 个客户端：  
  ```
  a 客户端：
  > del a
  (integer) 0
  127.0.0.1:6379> blpop a 10

  马上切换到 b 客户端：
  > lpush a hello
  (integer) 1

  切回 a 客户端，会看到：
  1) "a"
  2) "hello"
  (5.22s)
  ```

- ltrim key start stop  
  trim the list, retain items from start to stop  
  只保留 start 到 stop 的元素  
  示例
  ```
  > del lang
  (integer) 1
  127.0.0.1:6379> rpush lang Java Python PHP C++
  (integer) 4
  127.0.0.1:6379> ltrim lang 1 2
  OK
  127.0.0.1:6379> lrange lang 0 -1
  1) "Python"
  2) "PHP"
  ```

## hash

## set

## zset (sorted set)