# Redis 在电商系统中的应用

Redis作为高性能的内存数据库，在电商系统中扮演着至关重要的角色。它可以用于缓存、会话管理、购物车、限流、排行榜等多个场景，大幅提升系统性能和用户体验。

## 应用场景

### 1. 缓存层

#### 商品信息缓存
```redis
# 缓存商品基本信息
HSET product:1001 name "iPhone 15 Pro" price 7999 stock 100 sales 5000
EXPIRE product:1001 3600

# 批量获取商品信息
HMGET product:1001 name price stock

# 缓存商品详情（JSON格式）
SET product:detail:1001 '{"id":1001,"name":"iPhone 15 Pro","specs":{...}}' EX 3600
```

#### 分类信息缓存
```redis
# 缓存分类树
SET category:tree '{"id":1,"name":"电子产品","children":[...]}' EX 7200

# 缓存分类下的商品ID列表
ZADD category:products:100 1 1001 2 1002 3 1003
```

#### 热点数据缓存
```redis
# 首页推荐商品
LPUSH homepage:recommend 1001 1002 1003 1004 1005
EXPIRE homepage:recommend 1800

# 秒杀商品列表
ZADD seckill:products 1702800000 "product:5001" 1702803600 "product:5002"
```

### 2. 会话管理

#### 用户Session
```redis
# 存储用户会话信息
HSET session:abc123def456 user_id 10001 username "zhangsan" login_time 1702800000
EXPIRE session:abc123def456 7200

# 获取会话信息
HGETALL session:abc123def456

# 续期会话
EXPIRE session:abc123def456 7200

# 清除会话（登出）
DEL session:abc123def456
```

#### 单点登录（SSO）
```redis
# 用户登录token
SET token:user:10001 "abc123def456" EX 86400

# 检查用户是否在线
EXISTS token:user:10001

# 强制下线
DEL token:user:10001
```

### 3. 购物车

#### 购物车实现
```redis
# 添加商品到购物车（Hash存储）
HSET cart:user:10001 sku:1001 2 sku:1002 1

# 修改商品数量
HINCRBY cart:user:10001 sku:1001 1

# 删除购物车商品
HDEL cart:user:10001 sku:1001

# 获取购物车所有商品
HGETALL cart:user:10001

# 获取购物车商品数量
HLEN cart:user:10001

# 清空购物车
DEL cart:user:10001
```

#### 购物车持久化
```redis
# 设置购物车过期时间（30天）
EXPIRE cart:user:10001 2592000

# 购物车定时同步到数据库
# 通过定时任务扫描即将过期的购物车，同步到MySQL
```

### 4. 库存扣减

#### 预减库存（防止超卖）
```redis
# 初始化商品库存
SET stock:sku:1001 1000

# 扣减库存（原子操作）
DECR stock:sku:1001

# 检查库存是否充足
GET stock:sku:1001

# 回滚库存（取消订单）
INCR stock:sku:1001

# 使用Lua脚本保证原子性
EVAL "
local stock = redis.call('GET', KEYS[1])
if tonumber(stock) >= tonumber(ARGV[1]) then
    redis.call('DECRBY', KEYS[1], ARGV[1])
    return 1
else
    return 0
end
" 1 stock:sku:1001 5
```

#### 库存预警
```redis
# 设置库存告警阈值
SET stock:alert:sku:1001 50

# 检查库存是否低于阈值
# 在扣减库存时同时检查
```

### 5. 限流与防刷

#### 接口限流（令牌桶算法）
```redis
# 用户访问限流（每分钟最多100次）
INCR rate:limit:user:10001:202312071500
EXPIRE rate:limit:user:10001:202312071500 60

# 检查是否超过限制
GET rate:limit:user:10001:202312071500
```

#### IP限流
```redis
# IP访问频率限制
INCR rate:limit:ip:192.168.1.100:202312071500
EXPIRE rate:limit:ip:192.168.1.100:202312071500 60
```

#### 防刷机制
```redis
# 记录用户行为（滑动窗口）
ZADD user:actions:10001 1702800000 "action1" 1702800001 "action2"
ZREMRANGEBYSCORE user:actions:10001 0 1702799940  # 删除60秒前的记录
ZCARD user:actions:10001  # 统计最近60秒的操作次数
```

### 6. 秒杀系统

#### 秒杀库存
```redis
# 设置秒杀商品库存
SET seckill:stock:5001 1000

# 用户抢购
WATCH seckill:stock:5001
GET seckill:stock:5001
MULTI
DECR seckill:stock:5001
EXEC

# 使用Lua脚本实现更高效的秒杀
EVAL "
local stock = redis.call('GET', KEYS[1])
if tonumber(stock) > 0 then
    redis.call('DECR', KEYS[1])
    redis.call('SADD', KEYS[2], ARGV[1])
    return 1
else
    return 0
end
" 2 seckill:stock:5001 seckill:users:5001 10001
```

#### 秒杀资格
```redis
# 记录已抢购用户（防止重复抢购）
SADD seckill:users:5001 10001

# 检查用户是否已抢购
SISMEMBER seckill:users:5001 10001

# 获取已抢购人数
SCARD seckill:users:5001
```

### 7. 排行榜

#### 商品销量排行
```redis
# 增加商品销量
ZINCRBY product:sales:ranking 10 "product:1001"

# 获取销量前10的商品
ZREVRANGE product:sales:ranking 0 9 WITHSCORES

# 获取某商品排名
ZREVRANK product:sales:ranking "product:1001"
```

#### 用户积分排行
```redis
# 增加用户积分
ZINCRBY user:points:ranking 100 "user:10001"

# 获取积分前100的用户
ZREVRANGE user:points:ranking 0 99 WITHSCORES

# 获取用户排名
ZREVRANK user:points:ranking "user:10001"
```

### 8. 分布式锁

#### 商品秒杀锁
```redis
# 获取锁
SET lock:seckill:5001 "random_value" NX EX 10

# 释放锁（Lua脚本保证原子性）
EVAL "
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
else
    return 0
end
" 1 lock:seckill:5001 "random_value"
```

#### Redlock 算法
```python
# 使用Redlock实现更可靠的分布式锁
# 在多个Redis实例上同时获取锁
```

### 9. 消息队列

#### 订单异步处理
```redis
# 发布订单消息
LPUSH queue:order:create '{"order_id":1001,"user_id":10001}'

# 消费订单消息
BRPOP queue:order:create 0

# 使用Stream实现更强大的消息队列
XADD orders:stream * order_id 1001 user_id 10001
XREAD STREAMS orders:stream 0
```

#### 延迟队列
```redis
# 使用ZSet实现延迟队列
ZADD delay:queue 1702800600 "task:cancel:order:1001"

# 定时扫描到期任务
ZRANGEBYSCORE delay:queue 0 1702800000
```

### 10. 计数器

#### 商品浏览量
```redis
# 增加浏览量
INCR product:views:1001

# 批量增加
INCRBY product:views:1001 10

# 获取浏览量
GET product:views:1001
```

#### 优惠券剩余数量
```redis
# 初始化优惠券数量
SET coupon:count:C001 10000

# 领取优惠券
DECR coupon:count:C001

# 检查剩余数量
GET coupon:count:C001
```

### 11. 地理位置

#### 附近门店
```redis
# 添加门店位置
GEOADD stores 116.404 39.915 "store:1001"
GEOADD stores 116.405 39.916 "store:1002"

# 查找附近5公里的门店
GEORADIUS stores 116.404 39.915 5 km WITHDIST

# 计算两个门店之间的距离
GEODIST stores "store:1001" "store:1002" km
```

### 12. 布隆过滤器

#### 防止缓存穿透
```redis
# 使用RedisBloom模块
BF.ADD product:bloom 1001
BF.ADD product:bloom 1002

# 检查商品是否存在
BF.EXISTS product:bloom 1001

# 批量检查
BF.MEXISTS product:bloom 1001 1002 9999
```

## 缓存策略

### 缓存更新策略

#### Cache Aside（旁路缓存）
```python
def get_product(product_id):
    # 1. 先查缓存
    product = redis.get(f"product:{product_id}")
    if product:
        return product
    
    # 2. 缓存未命中，查数据库
    product = db.query("SELECT * FROM products WHERE product_id = ?", product_id)
    
    # 3. 写入缓存
    redis.setex(f"product:{product_id}", 3600, product)
    
    return product

def update_product(product_id, data):
    # 1. 更新数据库
    db.update("UPDATE products SET ... WHERE product_id = ?", product_id)
    
    # 2. 删除缓存
    redis.delete(f"product:{product_id}")
```

#### Write Through（写穿）
```python
def update_product(product_id, data):
    # 1. 先更新缓存
    redis.setex(f"product:{product_id}", 3600, data)
    
    # 2. 同步更新数据库
    db.update("UPDATE products SET ... WHERE product_id = ?", product_id)
```

### 缓存雪崩防护

```python
import random

# 随机过期时间
expire_time = 3600 + random.randint(0, 300)
redis.setex(key, expire_time, value)

# 永不过期 + 异步更新
redis.set(key, value)
# 使用后台任务定期更新热点数据
```

### 缓存击穿防护

```python
import threading

# 互斥锁防止缓存击穿
lock = threading.Lock()

def get_hot_product(product_id):
    product = redis.get(f"product:{product_id}")
    if product:
        return product
    
    # 获取锁
    with lock:
        # 双重检查
        product = redis.get(f"product:{product_id}")
        if product:
            return product
        
        # 查询数据库
        product = db.query("SELECT * FROM products WHERE product_id = ?", product_id)
        
        # 写入缓存
        redis.setex(f"product:{product_id}", 3600, product)
        
        return product
```

### 缓存穿透防护

```python
# 1. 空值缓存
def get_product(product_id):
    product = redis.get(f"product:{product_id}")
    if product == "null":
        return None
    if product:
        return product
    
    product = db.query("SELECT * FROM products WHERE product_id = ?", product_id)
    
    if product:
        redis.setex(f"product:{product_id}", 3600, product)
    else:
        # 缓存空值，但设置较短的过期时间
        redis.setex(f"product:{product_id}", 60, "null")
    
    return product

# 2. 布隆过滤器
def get_product(product_id):
    # 先用布隆过滤器判断
    if not bf.exists(f"product:{product_id}"):
        return None
    
    # 再查缓存和数据库
    ...
```

## 高可用方案

### 主从复制
```bash
# 从节点配置
replicaof master-ip master-port
```

### 哨兵模式
```bash
# 哨兵配置
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 10000
```

### 集群模式
```bash
# 创建集群
redis-cli --cluster create \
  127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 \
  127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
  --cluster-replicas 1
```

## 性能优化

### 连接池配置
```python
from redis import ConnectionPool, Redis

pool = ConnectionPool(
    host='localhost',
    port=6379,
    db=0,
    max_connections=100,
    decode_responses=True
)

redis_client = Redis(connection_pool=pool)
```

### Pipeline批量操作
```python
# 使用pipeline减少网络往返
pipe = redis_client.pipeline()
for i in range(1000):
    pipe.hset(f"product:{i}", "views", 0)
pipe.execute()
```

### 使用Hash存储对象
```python
# Hash比String更节省内存
redis.hset("product:1001", mapping={
    "name": "iPhone 15",
    "price": "7999",
    "stock": "100"
})
```

## 监控指标

1. **内存使用率**：防止内存溢出
2. **命中率**：缓存命中率应该在90%以上
3. **QPS**：每秒查询数
4. **慢查询**：定期检查慢查询日志
5. **键数量**：控制键的总数
6. **过期键数量**：合理设置过期时间

## 最佳实践

1. **合理设置过期时间**：避免数据永久驻留内存
2. **避免大key**：单个key不要超过10KB
3. **使用批量操作**：减少网络往返次数
4. **选择合适的数据结构**：根据场景选择最优数据结构
5. **监控内存使用**：设置内存淘汰策略
6. **定期备份**：开启AOF和RDB持久化
7. **安全防护**：设置密码、禁用危险命令

## 参考链接

- [Redis官方文档](https://redis.io/documentation)
- [Redis实践](../../03~键值型数据库/Redis/README.md)
