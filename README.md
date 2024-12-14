
# 参考：


* [https://stackoverflow.com/questions/73242557/jedispool\-vs\-jedispooled](https://github.com)
* [https://redis.io/docs/latest/develop/clients/jedis/](https://github.com)
* [https://support.huaweicloud.com/intl/zh\-cn/dcs\_faq/dcs\-faq\-211230001\.html](https://github.com):[wgetcloud全球加速器服务](https://wgetcloud6.org)
* [https://www.site24x7\.com/blog/jedis\-pool\-optimization](https://github.com)


# Jedis


Jedis 是一个 Java 客户端，用于与 Redis 数据库进行交互。它提供了一系列简单易用的 API，使得在 Java 应用程序中使用 Redis 变得非常方便。以下是 Jedis 的使用方法及一些注意事项。


### Jedis的优势


Lettuce客户端及Jedis客户端比较如下：


* Lettuce：
	+ Lettuce客户端没有连接保活探测，错误连接存在连接池中会造成请求超时报错。
	+ Lettuce客户端未实现testOnBorrow等连接池检测方法，无法在使用连接之前进行连接校验。
* Jedis：
	+ Jedis客户端实现了testOnBorrow、testWhileIdle、testOnReturn等连接池校验配置。
	+ 开启testOnBorrow在每次借用连接前都会进行连接校验，可靠性最高，但是会影响性能（每次Redis请求前会进行探测）。
	+ testWhileIdle可以在连接空闲时进行连接检测，合理配置阈值可以及时剔除连接池中的异常连接，防止使用异常连接造成业务报错。
	+ 在空闲连接检测之前，连接出现问题，可能会造成使用该连接的业务报错，此处可以通过参数控制检测间隔（timeBetweenEvictionRunsMillis）。


因此，Jedis客户端在面对连接异常，网络抖动等场景下的异常处理和检测能力明显强于Lettuce，可靠性更强。


### Jedis 使用


#### 1\. 引入依赖


如果你使用 Maven，可以在 `pom.xml` 中添加以下依赖：



```

    redis.clients
    jedis
    5.2.0 


```

#### 2\. 创建 Jedis 实例


创建一个 Jedis 实例，连接到 Redis 服务器：



```
import redis.clients.jedis.Jedis;

public class JedisExample {
    public static void main(String[] args) {
        // 创建一个 Jedis 实例，连接到 localhost:6379
        Jedis jedis = new Jedis("localhost", 6379);
        
        // 进行身份验证（如果需要）
        // jedis.auth("your_password");

        // 测试连接
        System.out.println("连接成功: " + jedis.ping());
        
        // 关闭连接
        jedis.close();
    }
}

```

#### 3\. 常用操作


* **字符串操作**：



```
// 设置值
jedis.set("key", "value");
// 获取值
String value = jedis.get("key");
System.out.println("获取的值: " + value);

```

* **哈希操作**：



```
// 设置哈希
jedis.hset("user:1000", "name", "Alice");
jedis.hset("user:1000", "age", "30");

// 获取哈希
String name = jedis.hget("user:1000", "name");
System.out.println("用户姓名: " + name);

```

* **列表操作**：



```
// 添加元素到列表
jedis.lpush("mylist", "item1");
jedis.lpush("mylist", "item2");

// 获取列表元素
List list = jedis.lrange("mylist", 0, -1);
System.out.println("列表内容: " + list);

```

* **集合操作**：



```
// 添加元素到集合
jedis.sadd("myset", "member1");
jedis.sadd("myset", "member2");

// 获取集合成员
Set members = jedis.smembers("myset");
System.out.println("集合成员: " + members);

```

### 注意事项


1. **连接管理**：


	* 每次操作前创建和关闭 Jedis 实例会导致性能问题，建议使用连接池。
	* 可以使用 `JedisPool` 来管理连接。
	* Jedis可以使用tr\-with\-resources管理资源
```
import redis.clients.jedis.JedisPool;
import redis.clients.jedis.Jedis;

public class JedisPoolExample {
    public static void main(String[] args) {
      JedisPoolConfig config = new JedisPoolConfig();
       config.setMaxTotal(100);       // 最大连接数
       config.setMaxIdle(50);         // 最大空闲连接数
       config.setMinIdle(10);         // 最小空闲连接数
       config.setTestOnBorrow(true);  // 在获取连接时检查连接有效性
       config.setTestWhileIdle(true);  // 在空闲时检查连接有效性
       config.setMinEvictableIdleTimeMillis(60000); // 空闲连接最小存活时间，60S
       config.setTimeBetweenEvictionRunsMillis(30000); // 清理线程运行时间间隔，30S
       JedisPool pool = new JedisPool(config, "localhost", 6379);
        try (Jedis jedis = pool.getResource()) {
            System.out.println("连接成功: " + jedis.ping());
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            pool.close();
        }
    }
}

```
2. **异常处理**：


	* 在使用 Jedis 时，应当处理可能的异常，例如连接失败、超时等。
3. **线程安全**：


	* Jedis 实例不是线程安全的，因此不应在多个线程之间共享同一个实例。使用连接池可以避免这个问题。
4. **Redis 配置**：


	* 确保 Redis 服务正常运行，并根据需求调整 Redis 配置（如最大连接数、超时时间等）。
5. **数据过期**：


	* Redis 提供了键的过期功能，可以通过 `expire` 命令设置键的有效时间，以防止数据长时间占用内存。
6. **监控与优化**：


	* 监控 Redis 性能指标（如内存使用、命令执行时间等），并根据实际情况进行优化。


### 连接池推荐配置




| 参数 | 配置介绍 | 配置建议 |
| --- | --- | --- |
| maxTotal | 最大连接，单位：个 | 根据Web容器的Http线程数来进行配置，估算单个Http请求中可能会并行进行的Redis调用次数，例如：Tomcat中的Connector内的maxConnections配置为150，每个Http请求可能会并行执行2个Redis请求，在此之上进行部分预留，则建议配置至少为：150 x 2 \+ 100\= 400 限制条件：单个Redis实例的最大连接数。maxTotal和客户端节点数（CCE容器或业务VM数量）数值的乘积要小于单个Redis实例的最大连接数。例如：Redis主备实例配置maxClients为10000，单个客户端maxTotal配置为500，则最大客户端节点数量为20个。 |
| maxIdle | 最大空闲连接，单位：个 | 配置与maxTotal一致。 |
| minIdle | 最小空闲连接，单位：个 | 一般来说建议配置为maxTotal的X分之一，例如此处常规配置建议为：100。对于性能敏感的场景，为了防止经常连接数量抖动造成影响，可以配置与maxIdle一致，例如：400。 |
| maxWaitMillis | 最大获取连接等待时间，单位：毫秒 | 获取连接时最大的连接池等待时间，根据单次业务最长容忍的失败时间减去执行命令的超时时间得到建议值。例如：Http最长容忍的失败时间为15s，Redis请求的timeout设置为10s，则此处可以配置为5s。 |
| timeout | 命令执行超时时间，单位：毫秒 | 单次执行Redis命令最大可容忍的超时时间，根据业务程序的逻辑进行选择，出于对网络容错等考虑建议配置为不小于210ms。特殊的探测逻辑或者环境异常检测等，可以适当调整达到秒级。 |
| minEvictableIdleTimeMillis | 空闲连接逐出时间，大于该值的空闲连接一直未被使用则会被释放，单位：毫秒 | 如果希望系统不会经常对连接进行断链重建，此处可以配置一个较大值（xx分钟），或者此处配置为\-1并且搭配空闲连接检测进行定期检测。 |
| timeBetweenEvictionRunsMillis | 空闲连接探测时间间隔，单位：毫秒 | 根据系统的空闲连接数量进行估算，例如系统的空闲连接探测时间配置为30s，则代表每隔30s会对连接进行探测，如果30s内发生异常的连接，经过探测后会进行连接排除。根据连接数的多少进行配置，如果连接数太大，配置时间太短，会造成请求资源浪费。对于几百级别的连接，常规来说建议配置为30s，可以根据系统需要进行动态调整。 |
| testOnBorrow | 向资源池借用连接时是否做连接有效性检测（ping），检测到的无效连接将会被移除。 | 对于业务连接极端敏感的，并且性能可以接受的情况下，可以配置为True，一般来说建议配置为False，启用连接空闲检测。 |
| testWhileIdle | 是否在空闲资源监测时通过ping命令监测连接有效性，无效连接将被销毁。 | True |
| testOnReturn | 向资源池归还连接时是否做连接有效性检测（ping），检测到无效连接将会被移除。 | False |
| maxAttempts | 在JedisCluster模式下，您可以配置maxAttempts参数来定义失败时的重试次数。 | 建议配置3\-5之间，默认配置为5。根据业务接口最大超时时间和单次请求的timeout综合配置，最大配置不建议超过10，否则会造成单次请求处理时间过长，接口请求阻塞。 |


