# jedis

> Jedis 练手工程：Maven + JUnit 5，用一条测试演示连上 Redis 并操作 Hash 类型

## 项目简介

这是一个只有 7 个文件的最小 Maven 工程，用来练 Jedis（Redis 的 Java 客户端）的基本用法。

工程里**没有 `src/main`，只有 `src/test/java/com/heima/test/JedisTest.java` 一个类、38 行代码**。它做的事很简单：在 `@BeforeEach` 里连上一台 Redis，跑一个测试方法往里写两个 Hash 字段再整体读回来，最后 `@AfterEach` 关掉连接。

包名 `com.heima`（黑马）说明它是跟着课程敲的练习。因为代码量太小，这个 README 主要交代两件事：**它到底演示了哪些 Jedis API**（答案是比预想的少），以及**怎么把它跑起来**（需要一台能连的 Redis，且必须改掉写死的 IP）。

## 内容概览

### 唯一的测试类

`src/test/java/com/heima/test/JedisTest.java` 的完整结构：

| 部分 | 内容 | 说明 |
| --- | --- | --- |
| 字段 | `private Jedis jedis;` | 每次测试方法用一个新连接，不复用 |
| `@BeforeEach setUp()` | `jedis = new Jedis("10.0.12.205", 6379);` | **单机直连**：用 host + port 构造 `Jedis`，默认无密码、database 0 |
| `@Test test()` | `hset` ×2 → `hgetAll` → `println` | 只操作 **Hash** 一种数据类型 |
| `@AfterEach tearDown()` | `jedis.close();` 带 null 判断 | 归还/关闭连接，避免测试之间泄漏 |

测试方法体只有 5 行：

```java
jedis.hset("user:1", "name", "Jack");
jedis.hset("user:1", "age", "56");
Map<String, String> stringStringMap = jedis.hgetAll("user:1");
System.out.println(stringStringMap);
```

对应到 Redis 命令就是 `HSET user:1 name Jack`、`HSET user:1 age 56`、`HGETALL user:1`，key 用了 `user:1` 这种「业务前缀 + id」的常见命名。

### 覆盖了 / 没覆盖哪些 Redis 操作

实测只覆盖了下面这些：

| 能力 | 是否演示 | 依据 |
| --- | :-: | --- |
| 建立连接（`new Jedis(host, port)`） | ✅ | `setUp()` |
| 关闭连接（`close()`） | ✅ | `tearDown()` |
| Hash 类型：`hset` / `hgetAll` | ✅ | `test()` |
| String 类型（`set`/`get`/`incr`） | ❌ | 无 |
| List 类型（`lpush`/`lrange`） | ❌ | 无 |
| Set 类型（`sadd`/`smembers`） | ❌ | 无 |
| ZSet 类型（`zadd`/`zrange`） | ❌ | 无 |
| 事务（`multi`/`exec`/`Transaction`） | ❌ | 无 |
| 连接池（`JedisPool` / `JedisPooled`） | ❌ | 无 `commons-pool2` 依赖，也没有池化代码 |
| Pipeline / 发布订阅 / Lua 脚本 | ❌ | 无 |
| 密码认证（`auth`）、选库（`select`） | ❌ | 构造 `Jedis` 时没传 password/database |

也就是说，**「五种数据类型 + 事务 + 连接池」这一整套并没有出现在仓库里**，目前只到「连上 + 会写一个 Hash」的程度。要补齐，最省事的做法是在同一个类里按下面这个骨架继续加 `@Test` 方法：

```java
@Test void stringDemo() { jedis.set("k", "v"); System.out.println(jedis.get("k")); }

@Test void listDemo()   { jedis.rpush("list", "a", "b"); System.out.println(jedis.lrange("list", 0, -1)); }

@Test void setDemo()    { jedis.sadd("set", "a", "b"); System.out.println(jedis.smembers("set")); }

@Test void zsetDemo()   { jedis.zadd("rank", 100, "tom"); System.out.println(jedis.zrangeWithScores("rank", 0, -1)); }

@Test void txDemo() {
    Transaction tx = jedis.multi();
    tx.set("t1", "1");
    tx.incr("t1");
    System.out.println(tx.exec());   // 返回 List<Object>，事务内的命令一次性提交
}
```

连接池需要额外加 `org.apache.commons:commons-pool2` 依赖，然后：

```java
JedisPoolConfig cfg = new JedisPoolConfig();
cfg.setMaxTotal(20);
cfg.setMaxIdle(10);
try (JedisPool pool = new JedisPool(cfg, "127.0.0.1", 6379)) {
    try (Jedis j = pool.getResource()) {
        j.set("pool:key", "ok");
        System.out.println(j.get("pool:key"));
    }
}
```

## 技术栈与运行方式

### 依赖（来自 `pom.xml`）

| 坐标 | 版本 | scope | 用途 |
| --- | --- | --- | --- |
| `redis.clients:jedis` | **5.2.0** | compile（默认） | Redis 客户端，提供 `Jedis`、`Transaction`、`JedisPool` 等 |
| `org.junit.jupiter:junit-jupiter` | **5.7.0** | test | JUnit 5 聚合依赖（api + params + engine 一次引入） |

- `groupId` / `artifactId` / `version`：`com.heima` / `jedis` / `1.0-SNAPSHOT`
- 编译级别：`maven.compiler.source/target = 11`，源码编码 `UTF-8`
- **没有** `commons-pool2`，所以 `JedisPool` 相关 API 在这个工程里暂时用不了（Jedis 4+ 已把池化依赖做成可选）
- **没有** `maven-surefire-plugin` 配置，JUnit 5 需要 surefire 2.22+ 才能被自动发现，建议显式加一条 3.x 版本，否则 `mvn test` 有可能显示「Tests run: 0」

### JDK 版本

- `pom.xml` 声明 **Java 11**（`maven.compiler.source/target`），这是跑测试的实际下限。
- `.idea/misc.xml` 里写的是 `languageLevel="JDK_22"`，是 IDEA 打开时按本机 SDK 自动填的，与 pom 不一致——以 pom 的 11 为准即可，用 11 / 17 / 21 都能编译（Jedis 5.x 要求 Java 8+）。

### 运行前提：需要一台能连的 Redis

**需要。** 而且代码里写死的是 `10.0.12.205:6379`——一个内网地址，在别的机器上必然连不上，会抛 `JedisConnectionException: Failed to connect to any host resolved for DNS name`。

两种跑法，任选其一：

**① 用本地 Redis（推荐）**

```bash
# 起一个 Redis（Docker）
docker run -d --name redis-lab -p 6379:6379 redis:7-alpine
redis-cli -h 127.0.0.1 ping        # 期望输出 PONG
```

然后把 `JedisTest.java` 第 21 行改成：

```java
jedis = new Jedis("127.0.0.1", 6379);
```

**② 用远程/内网 Redis**：保留原 IP，确认本机能访问该地址的 6379 端口（`nc -vz 10.0.12.205 6379`）。如果有密码，还要改成 `new Jedis(host, port, user, password)` 或连接后 `jedis.auth("...")`。

### 跑测试

```bash
# 编译 + 跑测试（首次会下载 jedis 5.2.0 与 junit-jupiter 5.7.0）
mvn -q test

# 只跑这一个类
mvn -q test -Dtest=JedisTest

# 看输出（hgetAll 的结果是 println 出来的，surefire 默认会捕获）
cat target/surefire-reports/com.heima.test.JedisTest.txt
```

期望输出类似：`{name=Jack, age=56}`。用 IDEA 更直接：打开 `pom.xml` 让 IDEA 导入 Maven 工程，然后在 `JedisTest` 左边点绿色三角。

> 这个工程**没有可以 `main` 启动的入口**，因为它没有 `src/main/java`。想写个能直接 `java -cp` 跑的小 demo，得先补 `src/main/java/com/heima/Demo.java`。

## 目录结构

```
jedis/
├── README.md                                        # 本次补写
├── pom.xml                                          # Maven 坐标 + jedis 5.2.0 + junit-jupiter 5.7.0
├── .gitignore                                       # 忽略 target/、.idea/modules.xml、*.iml 等
├── .idea/                                           # 4 个文件，已入库
│   ├── misc.xml                                     # 语言级别 JDK_22（与 pom 的 11 不一致）
│   ├── vcs.xml
│   ├── inspectionProfiles/Project_Default.xml
│   └── .gitignore
└── src/
    └── test/
        └── java/com/heima/test/
        └── JedisTest.java                           # 唯一源文件，38 行
```

注意目录层级：`src/test/java/...`，**没有 `src/main/java`**，也没有 `src/test/resources`。

## 学习路线 / 说明

这个仓库是一次「最小可运行」的尝试，按它的现状，可以当作 Jedis 练习的起点，往下补的顺序建议是：

1. **连接管理**：先在 `@BeforeEach`/`@AfterEach` 里把「建连—关连接」跑通（现在这步已经有了），再试 `JedisPooled`（Jedis 4+ 推荐的线程安全单例写法）。
2. **五种数据类型**：String → List → Set → Hash（已有）→ ZSet，每种写一个 `@Test`，对照 Redis 命令行确认结果。
3. **事务与 Pipeline**：`multi()` / `exec()` 看返回值；再用 `Pipeline` 对比一次 RTT 差异。
4. **键与过期**：`expire` / `ttl` / `keys` / `scan`（生产环境用 `scan` 而不是 `keys`）。
5. **Spring Data Redis**：如果目标是 Spring Boot 项目，`RedisTemplate` / `StringRedisTemplate` 是下一步；Jedis 只是底层实现之一（另一条路是 Lettuce）。

## 已知问题

- **写死内网 IP `10.0.12.205:6379`**：换台机器 100% 连不上，且没有通过 `application.properties`、环境变量或系统属性注入的方式。这是最影响「别人 clone 下来能不能跑」的一点，建议改成 `System.getProperty("redis.host", "127.0.0.1")` 之类的形式。
- **没有 `src/main`**：整个工程只有测试代码，没有 `main` 方法，不能 `java -jar` 或 `java -cp` 直接演示，只能靠 `mvn test`。作为「练习」没问题，但作为给别人看的 demo 门槛偏高。
- **只覆盖 Hash 一种类型**：README 想讲的「五种数据类型、事务、连接池」在代码里都不存在（详见上面的覆盖表）。如果这个仓库要作为 Jedis 学习笔记公开，建议按「学习路线」补齐后再更新简介。
- **`@BeforeAll` 是未使用的 import**：第 4 行 `import org.junit.jupiter.api.BeforeAll;` 没有任何方法用到，属于写代码时的残留。
- **JUnit 5 + 默认 surefire 版本的风险**：pom 里没有 `maven-surefire-plugin`，Maven 默认绑定的 surefire 版本偏低时不会执行 JUnit 5 测试（表现为 `Tests run: 0`，不报错）。补一条 `<plugin>maven-surefire-plugin 3.2.5</plugin>` 更稳。
- **`.idea/` 已入库**：`.gitignore` 忽略了 `.idea/modules.xml`、`.idea/compiler.xml`、`*.iml`，但 `misc.xml`、`vcs.xml`、`inspectionProfiles/Project_Default.xml` 仍被提交，且 `misc.xml` 里的 JDK_22 与 pom 的 11 冲突。建议整个 `.idea/` 都忽略。
- **工程名与依赖名撞车**：`artifactId` 就叫 `jedis`，和依赖 `redis.clients:jedis` 同名，命令行/日志里排查依赖冲突时容易看混，改成 `jedis-demo` 之类更清楚。
- **没有测试隔离**：测试直接往 `user:1` 写数据且不清理（没有 `@AfterEach` 里的 `del`），重复跑不会失败（Hash 覆盖写），但会在 Redis 里留下脏 key；换成 List/ZSet 的练习就会因为重复 push 导致断言不稳。建议每个测试用独立 key 前缀，或在 `tearDown` 里 `jedis.del(...)`。
