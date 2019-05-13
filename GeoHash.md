# GeoHash
地理位置模块，可以实现"附近的人"等功能。


## geoadd
添加元素  
syntax: geoadd key longitude latitude member [longitude latitude member ...]
```
redis-cli> geoadd mcdonald 113.331315 23.111575 maidanglao1
(integer) 1
redis-cli> geoadd company 116.48105 39.996794 juejin
(integer) 1
redis-cli> geoadd company 116.514203 39.905409 ireader
(integer) 1
redis-cli> geoadd company 116.489033 40.007669 meituan
(integer) 1
redis-cli> geoadd company 116.562108 39.787602 jd
(integer) 1
redis-cli> geoadd company 116.334255 40.027400 xiaomi
(integer) 1
```

## geodist
元素间的距离  
syntax: geodist key member1 member2 [unit]  
单位可以是：m(米)、km(公里)、ml(英里)、ft(尺)
```
redis-cli> geodist company jd xiaomi km
"33.0047"
redis-cli> geodist company juejin ireader km
"10.5501"
```

## geopos
获取元素坐标  
syntax: geopos key member [member ...]
```
redis-cli> geopos company juejin
1) 1) "116.48104995489120483"
   2) "39.99679348858259686"
redis-cli> geopos company jd
1) 1) "116.56210631132125854"
   2) "39.78760295130235392"
```

## geohash
获取元素的位置的GeoHash编码值(base32)  
syntax: geohash key member [member ...]
```
redis-cli> geohash company jd
1) "wx4fk7jgtf0"
redis-cli> geohash company juejin
1) "wx4gd94yjn0"
```
获取的编码值可以去 http//geohash.org/${geohash} 定位  
如：http://geohash.org/wx4fk7jgtf0


## georadiusbymember
获取以某个元素为中心，在给定半径范围内的其他元素（包括自己）  
syntax: georadiusbymember key member radius unit [WITHCOORD] [WITHDIST] [WITHHASH] [COUNT count] [ASC|DESC] [STORE key] [STOREDIST key]
```
redis-cli> georadiusbymember company ireader 12 km 
1) "ireader"
2) "juejin"
3) "meituan"
redis-cli> georadiusbymember company ireader 12 km withdist
1) 1) "ireader"
   2) "0.0000"
2) 1) "juejin"
   2) "10.5501"
3) 1) "meituan"
   2) "11.5748"
redis-cli> georadiusbymember company ireader 12 km withdist desc
1) 1) "meituan"
   2) "11.5748"
2) 1) "juejin"
   2) "10.5501"
3) 1) "ireader"
   2) "0.0000"
redis-cli> georadiusbymember company ireader 12 km withdist count 2
1) 1) "ireader"
   2) "0.0000"
2) 1) "juejin"
   2) "10.5501"
```

## georadius
获取以某个坐标为中心，在给定半径范围内的元素  
参数基本跟 georadiusbymember 一样，只是把 member 换成坐标  
syntax: georadiusbymember key longitude latitude radius unit [WITHCOORD] [WITHDIST] [WITHHASH] [COUNT count] [ASC|DESC] [STORE key] [STOREDIST key]
```
redis-cli> georadius company 116.514202 39.9 12 km 
1) "ireader"
2) "juejin"
redis-cli> georadius company 116.514202 39.9 12 km  withdist
1) 1) "ireader"
   2) "0.6016"
2) 1) "juejin"
   2) "11.1309"
```

## 底层原理
&ensp;&ensp;使用GeoHash算法，将二维的经纬度映射到一维坐标，这样二维空间上靠近的两个点在一维坐标上也很接近。  
&ensp;&ensp;那么这个映射算法是怎样的？它将这个地球的表面展开，看做一个二维平面，然后用二刀法，切成4个小的二维平面，然后对这4个小平面进行编码，分别标记为00、01、10、11，对小的二维平面又继续用二刀法切分。  
&ensp;&ensp;不断切分下去，最后的小平面就是一个一个的小方格，切的越细，精度就越高。  
&ensp;&ensp;redis 使用一个52位的整数值对小方格进行编码，使用一个zset进行存储，value是元素名，score是52的编码值。

验证：
```
redis-cli> zrange company 0 -1 withscores
 1) "jd"
 2) "4069154033428715"
 3) "xiaomi"
 4) "4069880904286516"
 5) "ireader"
 6) "4069886008361398"
 7) "juejin"
 8) "4069887154388167"
 9) "meituan"
10) "4069887179083478"
```