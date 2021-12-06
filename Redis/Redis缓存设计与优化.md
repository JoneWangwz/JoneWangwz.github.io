# Redis缓存设计与优化

## 1.缓存设计

### 1.1 缓存穿透

缓存穿透是指查询一个根本不存在的数据，缓存层和存储层都不会命中，通常出于容错的考虑，如果从存储层查不到数据通常不会写入到缓存层。

缓存穿透会导致每次查询一个不存在的数据会导致，每次都会去存储层进行查找数据，从而使得缓存层保存后端存储的意义。

造成缓存穿透的基本原因有两个：

1.自身业务代码或者数据出现问题

2.一些恶意攻击、爬虫等造成大量空命中

缓存穿透的解决方案

#### 1.缓存空对象

```java
String get(String key) {
 // 从缓存中获取数据
 String cacheValue = cache.get(key);
 // 缓存为空
 if (StringUtils.isBlank(cacheValue)) {
 // 从存储中获取
 String storageValue = storage.get(key);
 cache.set(key, storageValue);
 // 如果存储数据为空， 需要设置一个过期时间(300秒)
 if (storageValue == null) {
 cache.expire(key, 60 * 5);
 }
 return storageValue;
 } else {
 // 缓存非空
 return cacheValue;
 }
 }
```

#### 2.布隆过滤器

对于恶意攻击，向服务器请求大量不存在的数据造成的缓存击穿，还可以利用布隆过滤器先过滤一遍，对于不存在的数据先过滤掉，不让请求向后端发送。**当布隆过滤器说某个值存在时，这个值不一定存在，当布隆过滤器说某个值不存在时，这个值一定不存在。**

布隆过滤器就是一个非常大的位数组和几个不一样的无偏hash函数。所谓无偏就是能够把元素的hash值算的比较均匀。

当向布隆过滤器新增加key时，会使用多个hash函数对key进行计算得到一个整数索引，然后对位数组长度进行取模运算得到一个位置，然后将这个位置的设为1，当所有的key的所有的hash并且取模以后的值都是为1的时候，就完成了add。

向布隆过滤器查询某个值是否存在时，首先将这个key进行多个hash函数的计算，然后一次取模，找到对应的位数组的位置，如果找到的位数组的位置的值都为1，但这并不意味着这个key就是一定存在的，也可能是其他key的取模后的值将这个位置改成了1， 如果这个位数组是比较稀疏的话，那么这个key存在的概率是比较大的，如果有一个位置的取值不是1，那么布隆过滤器就认为这个key并不存在，就会返回。 

这种方法适用于数据命中不高、数据相对固定、实时性低(通常是数据集较大)的应用场景，代码维护较复杂，但是缓存空间占用很少。

下面是布隆过滤器的使用

```java
public class RedisBloomFilter {

    public static void main(String[] args) {
        Config config = new Config();
        config.useSingleServer().setAddress("redis://10.2.8.3:6379");
        RedissonClient redissonClient = Redisson.create(config);
        RBloomFilter<String> filter = redissonClient.getBloomFilter("nameList");

        filter.tryInit(100000L, 0.03);
        filter.add("jone");

        System.out.println(filter.contains("hello"));
        System.out.println(filter.contains("jone"));
    }
}
---------------------------------------------------------
false
true
```

使用布隆过滤器首先需要把所有数据都放入到布隆过滤器，并且在增加数据的时候也要往布隆过滤器里面放。使用布隆过滤器不能删除数据，要想删除数据除非重启布隆过滤器。

```java
//初始化布隆过滤器
RBloomFilter<String> bloomFilter = redisson.getBloomFilter("nameList");
//初始化布隆过滤器：预计元素为100000000L,误差率为3%
bloomFilter.tryInit(100000000L,0.03);
        
//把所有数据存入布隆过滤器
void init(){
    for (String key: keys) {
        bloomFilter.put(key);
    }
}

String get(String key) {
    // 从布隆过滤器这一级缓存判断下key是否存在
    Boolean exist = bloomFilter.contains(key);
    if(!exist){
        return "";
    }
    // 从缓存中获取数据
    String cacheValue = cache.get(key);
    // 缓存为空
    if (StringUtils.isBlank(cacheValue)) {
        // 从存储中获取
        String storageValue = storage.get(key);
        cache.set(key, storageValue);
        // 如果存储数据为空， 需要设置一个过期时间(300秒)
        if (storageValue == null) {
            cache.expire(key, 60 * 5);
        }
        return storageValue;
    } else {
        // 缓存非空
        return cacheValue;
    }
}
```

### 1.2 缓存失效

由于大量缓存在同一时间同时失效，可能会导致大量请求击穿缓存层直接访问数据库，可能会造成数据库瞬间压力过大挂掉，对于这种情况，再批量增加缓存的时候，最好将这一批数据的过期时间设置为一段时间的内的随机时间。比如进行秒杀活动时，当把所有的商品的过期时间设置成同样的时间，那么当缓存失效以后，再进来大量请求就会造成缓存失效，但是如果设置的一段时间的随机过期时间，会缓解缓存失效的风险。

```java
String get(String key) {
    // 从缓存中获取数据
    String cacheValue = cache.get(key);
    // 缓存为空
    if (StringUtils.isBlank(cacheValue)) {
        // 从存储中获取
        String storageValue = storage.get(key);
        cache.set(key, storageValue);
        //设置一个过期时间(300到600之间的一个随机数)
        int expireTime = new Random().nextInt(300)  + 300;
        if (storageValue == null) {
            cache.expire(key, expireTime);
        }
        return storageValue;
    } else {
        // 缓存非空
        return cacheValue;
    }
}
```

### 1.3 缓存雪崩

缓存雪崩是指缓存层支撑不住或者宕掉以后，流量就会直接访问后端的存储层。

由于缓存层承载着大量的请求，有效的保护了存储层，但如果缓存层不能提供服务(比如超大并发，缓存层支撑不住，或者是由于缓存设计不好，类似大量请求访问bigkey，导致缓存能支撑的并发急剧下降)，于是大量请求都会打到存储层，存储层的调用量会激增，造成存储层可能发生宕机。

预防和解决缓存雪崩的问题，可以从下面三个方面入手：

1.保证缓存层服务高可用性，比如使用redis sentinel或redis cluster。

2.依赖隔离组件为后端限流熔断并降级。比如使用sentinel或hystrix限流降级组件。

比如服务降级，可以针对不同的数据采用不同的处理方式。当业务访问的是非核心数据(例如电商商品属性，用户信息等)，暂时停止直接从缓存中查询这些数据，而是直接返回预定义的默认降级信息、空值或错误指示信息；当业务应用访问的是核心数据(例如电商商品库存)，仍允许查询缓存，如果缓存失效，也可以继续通过数据库读取。

3.提前演练。在项目上线之前，演练缓存层宕机以后，应用以及后端的负载均衡情况及可能出现的问题，在此基础上做预案设定。

## 2.缓存优化

2.1 热点缓存key重建优化

开发人员使用“缓存+过期时间”的策略既可以加速数据读取，有保证数据的定时更新，这种模式基本可以满足大多数情况下的需求。但是有两个问题同时出现，就可能会出现比较严重的问题:

- 当前key是一个热点key(比如一个热点的娱乐新闻)，并发量非常大；
- 重建缓存不能在短时间内完成，可能涉及复杂的SQL、多次I/O，多个依赖等。

在缓存失效的瞬间，有大量的线程重建缓存，造成后端存储层压力增大，甚至可能导致系统崩溃。要解决这个问题就要避免使用大量线程同时重建缓存。

```java
String get(String key) {
    // 从Redis中获取数据
    String value = redis.get(key);
    // 如果value为空， 则开始重构缓存
    if (value == null) {
        // 只允许一个线程重建缓存， 使用nx， 并设置过期时间ex
        String mutexKey = "mutext:key:" + key;
        if (redis.set(mutexKey, "1", "ex 180", "nx")) {
             // 从数据源获取数据
            value = db.get(key);
            // 回写Redis， 并设置过期时间
            redis.setex(key, timeout, value);
            // 删除key_mutex
            redis.delete(mutexKey);
        }// 其他线程休息50毫秒后重试
        else {
            Thread.sleep(50);
            get(key);
        }
    }
    return value;
}
```

缓存和数据库双写不一致

1.双写不一致

![image-20211206193515152](C:/Users/wangzhao/AppData/Roaming/Typora/typora-user-images/image-20211206193515152.png)

2.读写不一致

![image-20211206194102609](C:/Users/wangzhao/AppData/Roaming/Typora/typora-user-images/image-20211206194102609.png)

1.对于并发几率很小的数据，这种问题几乎不用考虑，很少会发生缓存不一致问题，可以给缓存加上过期时间，每隔一段时间主动进行更新缓存。

2.就算是并发很高，但是业务上能够容忍短时间内的缓存和数据库的不一致的问题(例如商品的名称、商品分类菜单)，设置过期时间，依然可以解决掉绝大多数的业务场景。

3.如果不能容忍缓存和数据不一致的情况，可以加上读写锁保证并发读写或写写的时候按照顺序排好，读读的时候相当于无锁。

上面针对读多写少的情况加入缓存提高性能，对于读多写多的情况还不能容忍数据不一致的情况，那就没必要加入缓存，直接操作数据库就好。放入缓存的数据应该是实时性、一致性要求不是很高的数据，不要为了用缓存，同时有保证绝对的一致性而做大量过度的设计和控制，增加系统复杂性！

## 3.缓存清理

redis对于过期健的三种清除策略：

1.被动删除：当读/写一个已过期的key时，会触发惰性删除策略，直接删除掉这个过期的key

2.主动删除：由于惰性删除策略无法保证冷数据被及时删除，所以redis会定期删除掉一批已过期的key

3.当前已用内存超过maxmemory限定，触发主动清楚策略

