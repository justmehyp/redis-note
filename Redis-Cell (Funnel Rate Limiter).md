# Redis-Cell (Funnel Rate Limiter)

漏斗有一个入口一个出口，一旦水满了，就无法再往里面装水，如果装水的速度 <= 漏水的速度。那么可以一直往里面装水；否则漏斗满了，则需要等待。  
这也是漏斗限速算法的原理。

## Java 实现漏斗限速算法
```java
public class Funnel {
    private int capacity;  //漏斗容量
    private float rateInSeconds;  // 漏斗"漏水"的速度
    private int used = 0; // 漏斗中的水
    private long lastTimestamp = System.currentTimeMillis(); // 记录上一次调用的时间戳

    public Funnel(int capacity, float rateInSeconds) {
        this.capacity = capacity;
        this.rateInSeconds = rateInSeconds;
    }

    /**
     * 尝试看能不能再往漏斗装水
     * @return true or false
     */
    public boolean throttle() {
        makeSpace();
        return doThrottle();
    }

    private void makeSpace() {
        long now = System.currentTimeMillis();
        long deltaTimestamp = now - lastTimestamp;
        long delta = (long) (deltaTimestamp/1000 * rateInSeconds);
        used -= delta;
    }

    private boolean doThrottle() {
        if (capacity - used > 0) {
            used++;
            this.lastTimestamp = System.currentTimeMillis();
            return true;
        } else {
            return false;
        }
    }
}
```
使用
```java
class Main {
    public static void main(String[] args) {
        Funnel funnel = new Funnel(5, 1);
        for (int i = 0; i < 10; i++) {
            System.out.println(new Date() + ": " + funnel.throttle());
        }
    }
}
```
输出：
```
Sun May 12 21:28:31 CST 2019: true
Sun May 12 21:28:31 CST 2019: true
Sun May 12 21:28:31 CST 2019: true
Sun May 12 21:28:31 CST 2019: true
Sun May 12 21:28:31 CST 2019: true
Sun May 12 21:28:31 CST 2019: false
Sun May 12 21:28:31 CST 2019: false
Sun May 12 21:28:31 CST 2019: false
Sun May 12 21:28:31 CST 2019: false
Sun May 12 21:28:31 CST 2019: false
```
可以看到，漏斗的容量为 5，漏水速度为 1/s, 一秒钟内请求了10次，前 5 次成功了，后 5 次都失败。  
修改下休眠时间间隔为 0.5 秒：
```java
class Main {
    public static void main(String[] args) throws InterruptedException {
        Funnel funnel = new Funnel(5, 1);
        for (int i = 0; i < 10; i++) {
            System.out.println(new Date() + ": " + funnel.throttle());
            Thread.sleep(500);
        }
    }
}
```
输出：
```
Sun May 12 21:35:04 CST 2019: true
Sun May 12 21:35:05 CST 2019: true
Sun May 12 21:35:05 CST 2019: true
Sun May 12 21:35:06 CST 2019: true
Sun May 12 21:35:06 CST 2019: true
Sun May 12 21:35:07 CST 2019: false
Sun May 12 21:35:07 CST 2019: true
Sun May 12 21:35:08 CST 2019: false
Sun May 12 21:35:08 CST 2019: true
Sun May 12 21:35:09 CST 2019: false
```

## 分布式漏斗限速算法 -- Redis-Cell 
- 下载  
https://github.com/brandur/redis-cell/releases  
根据系统下载对应的包，比如mac：redis-cell-v0.2.4-x86_64-apple-darwin.tar.gz

- 解压
```
shell> tar zxvf redis-cell-v0.2.4-x86_64-apple-darwin.tar.gz
```

- 安装
```
shell> sudo mkdir -p /usr/local/redis-modules/redis-cell
shell> sudo mv libredis_cell.dylib  /usr/local/redis-modules/redis-cell/
```

- 加载
```
redis-server --loadmodule /usr/local/redis-modules/redis-cell/libredis_cell.dylib 
```

- 使用
```
syntax: cl.throttle <key> <max_burst> <count per period> <period> [<quantity>] 

示例：
CL.THROTTLE   id  15 30 60 1
                  ▲  ▲  ▲  ▲
                  |  |  |  └──── 每次倒进去多少水（默认为1）
                  |  └──┴─────── 漏水速度： 30  / 60 seconds
                  └───────────── 容量-1， 实际容量：16
                  
输出：
1) (integer) 0    // 0,成功；1，失败
2) (integer) 16   // 容量
3) (integer) 15   // 剩余容量
4) (integer) -1   // 如果失败，需要等待多久
5) (integer) 2    // 多久可以漏完水
```
