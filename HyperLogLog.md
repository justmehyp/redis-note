# HyperLogLog
可以用来近似计算去重后的元素个数(UV)  
HyperLogLog 在元素较少时采用稀疏矩阵(sparse)存储，元素多了自动转化为稠密矩阵(dense)。

## pfadd
syntax: pfadd key element [element ...]  
添加元素到一个 HyperLogLog 中
```
pfadd id 1 2 3 4
```

## pfcount
syntax: pfcount key [key ...]  
返回给定 HyperLogLog 的大小（近似基数 approximated cardinality）
```
pfcount id
```
如果参数有多个 key，pfcount 会合并这些 HyperLogLog，在内部生成一个临时 HyperLogLog 来存储合并后的结果。
```
pfcount id1 id2
效果相当于：
pfmerge tmp-id id1 id2
pfcount tmp-id
```

## pfmerge
syntax: pfmerge destkey sourcekey [sourcekey ...]  
合并多个 HyperLogLog 到目标 HyperLogLog
```
pfmerge tmp-id id1 id2
```
