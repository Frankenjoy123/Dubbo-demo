# 精尽 Dubbo 源码分析 —— 注册中心（一）之抽象 API



# 1. 概述

在 [《精尽 Dubbo 源码分析 —— 项目结构一览》「3.5 dubbo-registry」](http://svip.iocoder.cn/Dubbo/registry-api/#) 中，对 `dubbo-registry` **注册中心**这个大模块做了大体的介绍。那么从本文开始，分享注册中心的代码实现。

本文分享 `dubbo-registry-api` 模块，注册中心的抽象 API ，类结构如下图：

![类图](http://www.iocoder.cn/images/Dubbo/2018_08_01/01.png)

😈 整体比较易懂，笔者在这里先不介绍，胖友可以看完本文，回过头看看，自己是不是理解了？！

下面，我们按照从左到右的顺序，逐个分享。

# 2. RegistryFactory

[`com.alibaba.dubbo.registry.RegistryFactory`](https://github.com/YunaiV/dubbo/blob/7ece72959dd8c96e17fc240a2b22b40391265bcc/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/RegistryFactory.java) ，注册中心工厂**接口**，代码如下：

```
@SPI("dubbo")
public interface RegistryFactory {

    /**
     * 连接注册中心.
     * <p>
     * 连接注册中心需处理契约：<br>
     * 1. 当设置check=false时表示不检查连接，否则在连接不上时抛出异常。<br>
     * 2. 支持URL上的username:password权限认证。<br>
     * 3. 支持backup=10.20.153.10备选注册中心集群地址。<br>
     * 4. 支持file=registry.cache本地磁盘文件缓存。<br>
     * 5. 支持timeout=1000请求超时设置。<br>
     * 6. 支持session=60000会话超时或过期设置。<br>
     *
     * @param url 注册中心地址，不允许为空
     * @return 注册中心引用，总不返回空
     */
    @Adaptive({"protocol"})
    Registry getRegistry(URL url);

}
```

- RegistryFactory 是一个 Dubbo SPI 拓展接口。

- ```
  #getRegistry(url)
  ```

   

  方法，获得注册中心 Registry 对象。

  - 注意方法上注释的**处理契约**。
  - `@Adaptive({"protocol"})` 注解，Dubbo SPI 会自动实现 RegistryFactory$Adaptive 类，根据 `url.protocol` 获得对应的 RegistryFactory 实现类。例如，`url.protocol = zookeeper` 时，获得 ZookeeperRegistryFactory 实现类。

## 2.1 AbstractRegistryFactory

[`com.alibaba.dubbo.registry.support.AbstractRegistryFactory`](https://github.com/YunaiV/dubbo/blob/7ece72959dd8c96e17fc240a2b22b40391265bcc/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/support/AbstractRegistryFactory.java) ，实现 RegistryFactory 接口，RegistryFactory 抽象类，实现了 Registry 的**容器管理**。

### 2.1.1 属性

```
// The lock for the acquisition process of the registry
private static final ReentrantLock LOCK = new ReentrantLock();

/**
 * Registry 集合
 *
 * key：{@link URL#toServiceString()}
 */
// Registry Collection Map<RegistryAddress, Registry>
private static final Map<String, Registry> REGISTRIES = new ConcurrentHashMap<String, Registry>();
```

- `REGISTRIES` 静态属性，Registry 集合。
- `LOCK` 静态属性，锁，用于 `#destroyAll()` 和 `#getRegistry(url)` 方法，对 `REGISTRIES` 访问的竞争。

### 2.1.2 createRegistry

`#createRegistry(url)` **抽象**方法，创建 Registry 对象。代码如下：

```
/**
 * 创建 Registry 对象
 *
 * @param url 注册中心地址
 * @return Registry 对象
 */
protected abstract Registry createRegistry(URL url);
```

子类实现该方法，创建其对应的 Registry 实现类。例如，ZookeeperRegistryFactory 的该方法，创建 ZookeeperRegistry 对象。

### 2.1.3 getRegistry

[`#getRegistry(url)`](https://github.com/YunaiV/dubbo/blob/d7c9cec324901c8285e602fdd3256cc9f5586357/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/support/AbstractRegistryFactory.java#L96-L132) **实现**方法，获得注册中心 Registry 对象。优先从缓存中获取，否则进行创建。

- 🙂 实现比较易懂，点击链接查看，有代码注释。

### 2.1.4 destroyAll

[`#destroyAll()`](https://github.com/YunaiV/dubbo/blob/d7c9cec324901c8285e602fdd3256cc9f5586357/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/support/AbstractRegistryFactory.java#L65-L94) 方法，销毁所有 Registry 对象。

- 🙂 实现比较易懂，点击链接查看，有代码注释。

# 3. RegistryService

[`com.alibaba.dubbo.registry.RegistryService`](https://github.com/YunaiV/dubbo/blob/f2458c11b045f85f654ed1719c75f9b0ba6397fe/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/RegistryService.java) ，注册中心服务**接口**，定义了注册、订阅、查询**三种**操作方法，如下：

- ```
  #register(url)
  ```

   

  方法，

  注册

  数据，比如：提供者地址，消费者地址，路由规则，覆盖规则，等数据。

  - `#unregister(url)` 方法，取消注册。

- `#subscribe(url, NotifyListener)` 方法，**订阅**符合条件的已注册数据，当有注册数据变更时自动推送。

  - `#unsubscribe(url, NotifyListener)` 方法，取消订阅。

  - 在

     

    ```
    URL.parameters.category
    ```

     

    属性上，表示订阅的数据分类。目前有四种类型：

    - `consumers` ，服务消费者列表。
    - `providers` ，服务提供者列表。
    - `routers` ，[路由规则](http://dubbo.apache.org/zh-cn/docs/user/demos/routing-rule.html)列表。
    - `configurations` ，[配置规则](http://dubbo.apache.org/zh-cn/docs/user/demos/config-rule.html)列表。

- `#lookup(url)` 方法，**查询**符合条件的已注册数据，与订阅的推模式相对应，这里为拉模式，只返回一次结果。

ps：注意方法上注释的处理契约。

## 3.1 Registry

[`com.alibaba.dubbo.registry.Registry`](https://github.com/YunaiV/dubbo/blob/f2458c11b045f85f654ed1719c75f9b0ba6397fe/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/Registry.java) ，注册中心**接口**。Registry 继承了：

- RegistryService 接口，拥有拥有注册、订阅、查询三种操作方法。
- [`com.alibaba.dubbo.common.Node`](https://github.com/YunaiV/dubbo/blob/f2458c11b045f85f654ed1719c75f9b0ba6397fe/dubbo-common/src/main/java/com/alibaba/dubbo/common/Node.java) 接口，拥有节点相关的方法。

## 3.2 AbstractRegistry

[`com.alibaba.dubbo.registry.support.AbstractRegistry`](http://svip.iocoder.cn/Dubbo/registry-api/) ，实现 Registry 接口，Registry 抽象类，实现了如下方法：

- 通用的注册、订阅、查询、通知等方法。
- 持久化注册数据到文件，以 properties 格式存储。应用于，重启时，无法从注册中心加载服务提供者列表等信息时，从该文件中读取。

### 3.2.1 属性

```
 1: // URL地址分隔符，用于文件缓存中，服务提供者URL分隔
 2: // URL address separator, used in file cache, service provider URL separation
 3: private static final char URL_SEPARATOR = ' ';
 4: // URL地址分隔正则表达式，用于解析文件缓存中服务提供者URL列表
 5: // URL address separated regular expression for parsing the service provider URL list in the file cache
 6: private static final String URL_SPLIT = "\\s+";
 7: 
 8: // Log output
 9: protected final Logger logger = LoggerFactory.getLogger(getClass());
10: /**
11:  *  本地磁盘缓存。
12:  *
13:  *  1. 其中特殊的 key 值 .registies 记录注册中心列表
14:  *  2. 其它均为 {@link #notified} 服务提供者列表
15:  */
16: // Local disk cache, where the special key value.registies records the list of registry centers, and the others are the list of notified service providers
17: private final Properties properties = new Properties();
18: /**
19:  * 注册中心缓存写入执行器。
20:  *
21:  * 线程数=1
22:  */
23: // File cache timing writing
24: private final ExecutorService registryCacheExecutor = Executors.newFixedThreadPool(1, new NamedThreadFactory("DubboSaveRegistryCache", true));
25: /**
26:  * 是否同步保存文件
27:  */
28: // Is it synchronized to save the file
29: private final boolean syncSaveFile;
30: /**
31:  * 数据版本号
32:  *
33:  * {@link #properties}
34:  */
35: private final AtomicLong lastCacheChanged = new AtomicLong();
36: /**
37:  * 已注册 URL 集合。
38:  *
39:  * 注意，注册的 URL 不仅仅可以是服务提供者的，也可以是服务消费者的
40:  */
41: private final Set<URL> registered = new ConcurrentHashSet<URL>();
42: /**
43:  * 订阅 URL 的监听器集合
44:  *
45:  * key：消费者的 URL ，例如消费者的 URL
46:  */
47: private final ConcurrentMap<URL, Set<NotifyListener>> subscribed = new ConcurrentHashMap<URL, Set<NotifyListener>>();
48: /**
49:  * 被通知的 URL 集合
50:  *
51:  * key1：消费者的 URL ，例如消费者的 URL ，和 {@link #subscribed} 的键一致
52:  * key2：分类，例如：providers、consumers、routes、configurators。【实际无 consumers ，因为消费者不会去订阅另外的消费者的列表】
53:  *            在 {@link Constants} 中，以 "_CATEGORY" 结尾
54:  */
55: private final ConcurrentMap<URL, Map<String, List<URL>>> notified = new ConcurrentHashMap<URL, Map<String, List<URL>>>();
56: /**
57:  * 注册中心 URL
58:  */
59: private URL registryUrl;
60: /**
61:  * 本地磁盘缓存文件，缓存注册中心的数据
62:  */
63: // Local disk cache file
64: private File file;
65: /**
66:  * 是否销毁
67:  */
68: private AtomicBoolean destroyed = new AtomicBoolean(false);
69: 
70: public AbstractRegistry(URL url) {
71:     setUrl(url);
72:     // Start file save timer
73:     syncSaveFile = url.getParameter(Constants.REGISTRY_FILESAVE_SYNC_KEY, false);
74:     // 获得 `file`
75:     String filename = url.getParameter(Constants.FILE_KEY, System.getProperty("user.home") + "/.dubbo/dubbo-registry-" + url.getParameter(Constants.APPLICATION_KEY) + "-" + url.getAddress() + ".cache");
76:     File file = null;
77:     if (ConfigUtils.isNotEmpty(filename)) {
78:         file = new File(filename);
79:         if (!file.exists() && file.getParentFile() != null && !file.getParentFile().exists()) {
80:             if (!file.getParentFile().mkdirs()) {
81:                 throw new IllegalArgumentException("Invalid registry store file " + file + ", cause: Failed to create directory " + file.getParentFile() + "!");
82:             }
83:         }
84:     }
85:     this.file = file;
86:     // 加载本地磁盘缓存文件到内存缓存
87:     loadProperties();
88:     // 通知监听器，URL 变化结果
89:     notify(url.getBackupUrls());
90: }
```

- `file` 属性，*见代码注释*。

- ```
  properties
  ```

   

  属性，

  见代码注释

  。

  - 数据流向
    - 启动时，从 `file` 读取数据到 `properties` 中。
    - 注册中心数据发生变更时，通知到 Registry 后，修改 `properties` 对应的值，并写入 `file` 。
  - 数据键值
    - 大多数情况下，键为服务消费者的 URL 的服务键( `URL#serviceKey()` )，对应的值为服务提供者列表、[路由规则](http://dubbo.apache.org/zh-cn/docs/user/demos/config-rule.html)列表、[配置规则](http://dubbo.apache.org/zh-cn/docs/user/demos/routing-rule.html)列表。
    - 特殊情况下，【TODO 8019】.registies
    - 因为值会存在为列表的情况，使用空格( `URL_SEPARATOR` ) 分隔。

- `syncSaveFile` 属性，`properties` 发生变更时候，是同步还是异步写入 `file` 。

- `registryCacheExecutor` 属性，*见代码注释*。

- ```
  lastCacheChanged
  ```

   

  属性，

  见代码注释

  。

  - 因为每次写入 `file` 是全量，而不是增量写入，通过版本号，避免老版本覆盖新版本。

- `registered` 属性，*见代码注释*。

- `subscribed` 属性，*见代码注释*。

- ```
  notified
  ```

   

  属性，

  见代码注释

  。

  - 从数据上，和 `properties` 比较相似。笔者认为有两点差异：1）数据格式上，`notified` 根据**分类**做了聚合；2）不从 `file` 中读取，都是从注册中心读取的数据。

- `registryUrl` 属性，*见代码注释*。

- `destroyed` 属性，*见代码注释*。

- 构造方法

  ，

  见代码注释

  。

  - 第 87 行：调用

     

    `#loadProperties()`

     

    方法，加载本地磁盘缓存文件到内存缓存。

    - 🙂 代码比较简单，点击链接查看。

  - 第 89 行：// 【TODO 8020】为什么构造方法，要通知，连监听器都没注册

### 3.2.2 register && unregister

- `#register(url)`
  - 从实现上，我们可以看出，并未向注册中心发起注册，仅仅是添加到 `registered` 中，进行状态的维护。实际上，真正的实现在 FailbackRegistry 类中。
- `#unregister(url)`
  - 和 `#register(url)` 的**处理方式**相同。

### 3.2.3 subscribe && unsubscribe

- `#subscribe(url, listener)`
  - 和 `#register(url)` 的**处理方式**相同。
- `#unsubscribe(url, listener)`
  - 和 `#register(url)` 的**处理方式**相同。

### 3.2.4 notify

[`#notify(url, listener, urls)`](https://github.com/YunaiV/dubbo/blob/06155d670fd1331fa1d2f41f7050338f9e9502c5/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/support/AbstractRegistry.java#L477-L535) 方法，通知监听器，URL 变化结果。这里我们有两点要注意下：

- 第一，向注册中心发起订阅后，会获取到**全量**数据，此时会被调用 `#notify(...)` 方法，即 Registry 获取到了全量数据。
- 第二，每次注册中心发生变更时，会调用 `#notify(...)` 方法，虽然变化是**增量**，调用这个方法的调用方，已经进行处理，传入的 `urls`依然是**全量**的。

代码如下：

```
 1: /**
 2:  * 通知监听器，URL 变化结果。
 3:  *
 4:  * 数据流向 `urls` => {@link #notified} => {@link #properties} => {@link #file}
 5:  *
 6:  * @param url 消费者 URL
 7:  * @param listener 监听器
 8:  * @param urls 通知的 URL 变化结果（全量数据）
 9:  */
10: protected void  notify(URL url, NotifyListener listener, List<URL> urls) {
11:     if (url == null) {
12:         throw new IllegalArgumentException("notify url == null");
13:     }
14:     if (listener == null) {
15:         throw new IllegalArgumentException("notify listener == null");
16:     }
17:     if ((urls == null || urls.isEmpty())
18:             && !Constants.ANY_VALUE.equals(url.getServiceInterface())) {
19:         logger.warn("Ignore empty notify urls for subscribe url " + url);
20:         return;
21:     }
22:     if (logger.isInfoEnabled()) {
23:         logger.info("Notify urls for subscribe url " + url + ", urls: " + urls);
24:     }
25:     // 将 `urls` 按照 `url.parameter.category` 分类，添加到集合
26:     Map<String, List<URL>> result = new HashMap<String, List<URL>>();
27:     for (URL u : urls) {
28:         if (UrlUtils.isMatch(url, u)) {
29:             String category = u.getParameter(Constants.CATEGORY_KEY, Constants.DEFAULT_CATEGORY);
30:             List<URL> categoryList = result.get(category);
31:             if (categoryList == null) {
32:                 categoryList = new ArrayList<URL>();
33:                 result.put(category, categoryList);
34:             }
35:             categoryList.add(u);
36:         }
37:     }
38:     if (result.size() == 0) {
39:         return;
40:     }
41:     // 获得消费者 URL 对应的在 `notified` 中，通知的 URL 变化结果（全量数据）
42:     Map<String, List<URL>> categoryNotified = notified.get(url);
43:     if (categoryNotified == null) {
44:         notified.putIfAbsent(url, new ConcurrentHashMap<String, List<URL>>());
45:         categoryNotified = notified.get(url);
46:     }
47:     // 处理通知的 URL 变化结果（全量数据）
48:     for (Map.Entry<String, List<URL>> entry : result.entrySet()) {
49:         String category = entry.getKey();
50:         List<URL> categoryList = entry.getValue();
51:         // 覆盖到 `notified`
52:         // 当某个分类的数据为空时，会依然有 urls 。其中 `urls[0].protocol = empty` ，通过这样的方式，处理所有服务提供者为空的情况。
53:         categoryNotified.put(category, categoryList);
54:         // 保存到文件
55:         saveProperties(url);
56:         // 通知监听器
57:         listener.notify(categoryList);
58:     }
59: }
```

- 第 25 至 37 行：将

   

  ```
  urls
  ```

   

  按照

   

  ```
  url.parameter.category
  ```

   

  分类，添加到集合

   

  ```
  result
  ```

   

  中。

  - 第 28 行：TODO 芋艿
  - 这里有一点要注意，每次传入的 `urls` 的“**全量**”，指的是至少要是**一个分类**的全量，而不一定是全部数据。

- 第 41 至 46 行：获得消费者 URL 对应的在 `notified` 中的数据。

- 第 47 至 58 行：按照

  分类

  ，循环处理通知的 URL 变化结果（全量数据）。

  - 第 51 至 53 行：将 `result` 覆盖到 `notified` 中。这里又有一点需要注意，当某个分类的数据为空时，会依然有 `urls` 。其中 `urls[0].protocol = empty` ，通过这样的方式，处理**所有服务提供者为空**的情况。

  - 第 55 行：调用

     

    `#saveProperties(url)`

     

    方法，保存到文件。

    - 🙂 代码比较简单，点击链接查看。

  - 第 57 行：调用 `NotifyListener#notify(urls)` 方法，通知监听器处理。例如，有新的服务提供者启动时，被通知，创建新的 Invoker 对象。

### 3.2.5 recover

- `#recover()`
  - 和 `#register(url)` 的**处理方式**相同。

在注册中心断开，重连成功，调用 `#recover()` 方法，进行恢复注册和订阅。

### 3.2.6 destroy

- `#destroy()`
  - 和 `#register(url)` 的**处理方式**相同。

在 JVM 关闭时，调用 `#destroy()` 方法，进行取消注册和订阅。

## 3.3 FailbackRegistry

[`com.alibaba.dubbo.registry.support.FailbackRegistry`](https://github.com/YunaiV/dubbo/blob/414a4799cef08f0b5263d838eeaf8d8f169f2cdc/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/support/AbstractRegistry.java) ，实现 AbstractRegistry 抽象类，支持失败重试的 Registry 抽象类。

在上文中的代码中，我们可以看到，AbstractRegistry 进行的注册、订阅等操作，更多的是修改状态，而无和注册中心实际的操作。FailbackRegistry 在 AbstractRegistry 的基础上，实现了和注册中心实际的操作，并且支持失败重试的特性。

### 3.3.1 属性

```
 1: /**
 2:  * 定时任务执行器
 3:  */
 4: // Scheduled executor service
 5: private final ScheduledExecutorService retryExecutor = Executors.newScheduledThreadPool(1, new NamedThreadFactory("DubboRegistryFailedRetryTimer", true));
 6: 
 7: /**
 8:  * 失败重试定时器，定时检查是否有请求失败，如有，无限次重试
 9:  */
10: // Timer for failure retry, regular check if there is a request for failure, and if there is, an unlimited retry
11: private final ScheduledFuture<?> retryFuture;
12: /**
13:  * 失败发起注册失败的 URL 集合
14:  */
15: private final Set<URL> failedRegistered = new ConcurrentHashSet<URL>();
16: /**
17:  * 失败取消注册失败的 URL 集合
18:  */
19: private final Set<URL> failedUnregistered = new ConcurrentHashSet<URL>();
20: /**
21:  * 失败发起订阅失败的监听器集合
22:  */
23: private final ConcurrentMap<URL, Set<NotifyListener>> failedSubscribed = new ConcurrentHashMap<URL, Set<NotifyListener>>();
24: /**
25:  * 失败取消订阅失败的监听器集合
26:  */
27: private final ConcurrentMap<URL, Set<NotifyListener>> failedUnsubscribed = new ConcurrentHashMap<URL, Set<NotifyListener>>();
28: /**
29:  * 失败通知通知的 URL 集合
30:  */
31: private final ConcurrentMap<URL, Map<NotifyListener, List<URL>>> failedNotified = new ConcurrentHashMap<URL, Map<NotifyListener, List<URL>>>();
32: /**
33:  * 是否销毁
34:  */
35: private AtomicBoolean destroyed = new AtomicBoolean(false);
36: 
37: public FailbackRegistry(URL url) {
38:     super(url);
39:     // 重试频率，单位：毫秒
40:     int retryPeriod = url.getParameter(Constants.REGISTRY_RETRY_PERIOD_KEY, Constants.DEFAULT_REGISTRY_RETRY_PERIOD);
41:     // 创建失败重试定时器
42:     this.retryFuture = retryExecutor.scheduleWithFixedDelay(new Runnable() {
43:         public void run() {
44:             // Check and connect to the registry
45:             try {
46:                 retry();
47:             } catch (Throwable t) { // Defensive fault tolerance
48:                 logger.error("Unexpected error occur at failed retry, cause: " + t.getMessage(), t);
49:             }
50:         }
51:     }, retryPeriod, retryPeriod, TimeUnit.MILLISECONDS);
52: }
```

- `retryExecutor` 属性，*见代码注释*。

- ```
  retryFuture
  ```

   

  属性，

  见代码注释

  。

  - 第 41 至 51 行，在构造方法中创建该定时器，在其 `#run()` 方法中，会调用 `#retry()` 方法，进行重试。

- ```
  failedXXX
  ```

   

  属性，

  见代码注释

  。

  - 每种操作都有一个记录失败的集合。

- `destroyed` 属性，*见代码注释*。

### 3.3.2 register && unregister

- [`#register(url)`](https://github.com/YunaiV/dubbo/blob/414a4799cef08f0b5263d838eeaf8d8f169f2cdc/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/support/FailbackRegistry.java#L162-L199)
- [`#unregister(url)`](https://github.com/YunaiV/dubbo/blob/414a4799cef08f0b5263d838eeaf8d8f169f2cdc/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/support/FailbackRegistry.java#L201-L238)

> 代码比较易懂，点击链接查看。

### 3.3.3 subscribe && unsubscribe

- [`#subscribe(url, listener)`](https://github.com/YunaiV/dubbo/blob/414a4799cef08f0b5263d838eeaf8d8f169f2cdc/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/support/FailbackRegistry.java#L240-L282)
- [`#unsubscribe(url, listener)`](https://github.com/YunaiV/dubbo/blob/414a4799cef08f0b5263d838eeaf8d8f169f2cdc/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/support/FailbackRegistry.java#L284-L324)

> 代码比较易懂，点击链接查看。

### 3.3.4 notify

- [`#notify(url, listener, url)`](https://github.com/YunaiV/dubbo/blob/414a4799cef08f0b5263d838eeaf8d8f169f2cdc/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/support/FailbackRegistry.java#L326-L352)

> 代码比较易懂，点击链接查看。

### 3.3.5 recover

- [`#recover()`](https://github.com/YunaiV/dubbo/blob/414a4799cef08f0b5263d838eeaf8d8f169f2cdc/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/support/FailbackRegistry.java#L354-L379) 方法，**完全覆盖父类方法**( 即不像前面几个方法，会调用父类的方法 )，将需要注册和订阅的 URL 添加到 `failedRegistered` `failedSubscribed` 属性中。这样，在 `#retry()` 方法中，会重试进行连接。

> 代码比较易懂，点击链接查看。

### 3.3.6 retry

- [`#retry()`](https://github.com/YunaiV/dubbo/blob/414a4799cef08f0b5263d838eeaf8d8f169f2cdc/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/support/FailbackRegistry.java#L381-L528) 方法，遍历五个 `failedXXX` 属性，重试对应的操作。

> 代码比较易懂，点击链接查看。

### 3.3.7 destroy

- [`#destroy()`](https://github.com/YunaiV/dubbo/blob/414a4799cef08f0b5263d838eeaf8d8f169f2cdc/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/support/FailbackRegistry.java#L530-L541) 方法，取消注册和订阅，并关闭定时器。

> 代码比较易懂，点击链接查看。

# 4. NotifyListener

[`com.alibaba.dubbo.registry.NotifyListener`](https://github.com/YunaiV/dubbo/blob/da5ebc2737d560dc0fe308793780695c6afc5fda/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/NotifyListener.java) ，通知监听器。当收到服务变更通知时触发，代码如下：

```
public interface NotifyListener {

    /**
     * 当收到服务变更通知时触发。
     * <p>
     * 通知需处理契约：<br>
     * 1. 总是以服务接口和数据类型为维度全量通知，即不会通知一个服务的同类型的部分数据，用户不需要对比上一次通知结果。<br>
     * 2. 订阅时的第一次通知，必须是一个服务的所有类型数据的全量通知。<br>
     * 3. 中途变更时，允许不同类型的数据分开通知，比如：providers, consumers, routers, overrides，允许只通知其中一种类型，但该类型的数据必须是全量的，不是增量的。<br>
     * 4. 如果一种类型的数据为空，需通知一个empty协议并带category参数的标识性URL数据。<br>
     * 5. 通知者(即注册中心实现)需保证通知的顺序，比如：单线程推送，队列串行化，带版本对比。<br>
     *
     * @param urls 已注册信息列表，总不为空，含义同{@link com.alibaba.dubbo.registry.RegistryService#lookup(URL)}的返回值。
     */
    void notify(List<URL> urls);

}
```

- 注意看方法上的注释，特别是**全量**、**分类**、**为空**、**顺序**。

NotifyListener 的子类如下图：

![类图](http://www.iocoder.cn/images/Dubbo/2018_08_01/02.png)

# 5. ProviderConsumerRegTable

[`com.alibaba.dubbo.registry.support.ProviderConsumerRegTable`](https://github.com/YunaiV/dubbo/blob/703764e6e82a72ddca7401c9302781e69c453f8e/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/support/ProviderConsumerRegTable.java) ，服务提供者和消费者注册表，存储 JVM 进程内**自己**的服务提供者和消费者的 Invoker 。

该信息用于 [Dubbo QOS](http://dubbo.apache.org/zh-cn/docs/user/references/qos.html) 使用，例如将 JVM 进程中，**自己**的服务提供者下线，又或者查询自己的服务提供者和消费者列表。

- [《Dubbo 用户指南 —— 在线运维命令 - QOS》](http://dubbo.apache.org/zh-cn/docs/user/references/qos.html)
- 后续会有文章分享 QOS ，本文不多啰嗦。

代码如下：

```
public class ProviderConsumerRegTable {

    /**
     * 服务提供者 Invoker 集合
     *
     * key：服务提供者 URL 服务键
     */
    public static ConcurrentHashMap<String, Set<ProviderInvokerWrapper>> providerInvokers = new ConcurrentHashMap<String, Set<ProviderInvokerWrapper>>();
    /**
     * 服务消费者 Invoker 集合
     *
     * key：服务消费者 URL 服务键
     */
    public static ConcurrentHashMap<String, Set<ConsumerInvokerWrapper>> consumerInvokers = new ConcurrentHashMap<String, Set<ConsumerInvokerWrapper>>();
    
    // .... 省略方法
    
}
```

- 如下方法，已经添加代码注释，胖友点击查看。
- 服务提供者
  - [`#registerProvider(invoker, registryUrl, providerUrl)`](https://github.com/YunaiV/dubbo/blob/703764e6e82a72ddca7401c9302781e69c453f8e/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/support/ProviderConsumerRegTable.java#L51-L70) 静态方法，注册 Provider Invoker 。
  - [`#getProviderInvoker(serviceUniqueName)`](https://github.com/YunaiV/dubbo/blob/703764e6e82a72ddca7401c9302781e69c453f8e/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/support/ProviderConsumerRegTable.java#L72-L84) 静态静态，获得指定服务键的 Provider Invoker 集合。
  - [`#getProviderWrapper(invoker)`](https://github.com/YunaiV/dubbo/blob/703764e6e82a72ddca7401c9302781e69c453f8e/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/support/ProviderConsumerRegTable.java#L86-L112) 静态方法，获得服务提供者对应的 Invoker Wrapper 对象。
- 服务消费者
  - [`#registerConsumer(invoker, registryUrl, consumerUrl, registryDirectory)`](https://github.com/YunaiV/dubbo/blob/703764e6e82a72ddca7401c9302781e69c453f8e/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/support/ProviderConsumerRegTable.java#L114-L134) 静态方法，注册 Consumer Invoker 。
  - [`#getConsumerInvoker(serviceUniqueName)`](https://github.com/YunaiV/dubbo/blob/703764e6e82a72ddca7401c9302781e69c453f8e/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/support/ProviderConsumerRegTable.java#L136-L148) 静态方法，获得指定服务键的 Consumer Invoker 集合。

## 5.1 ProviderInvokerWrapper

[`com.alibaba.dubbo.registry.support.ProviderInvokerWrapper`](https://github.com/YunaiV/dubbo/blob/703764e6e82a72ddca7401c9302781e69c453f8e/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/support/ProviderInvokerWrapper.java) ，实现 Invoker 接口，服务提供者 Invoker Wrapper ，代码如下：

```
/**
 * Invoker 对象
 */
private Invoker<T> invoker;
/**
 * 原始 URL
 */
private URL originUrl;
/**
 * 注册中心 URL
 */
private URL registryUrl;
/**
 * 服务提供者 URL
 */
private URL providerUrl;
/**
 * 是否注册
 */
private volatile boolean isReg;
    
// ... 省略方法
```

- 相比纯粹的 Invoker 对象，又多了运维命令需要的属性。例如 `isReg` **状态**属性，可以在使用**下线服务命令**后，标记为 `false` 。想提前深入了解的胖友，可以看下 [`com.alibaba.dubbo.qos.command.impl.Offline`](https://github.com/YunaiV/dubbo/blob/703764e6e82a72ddca7401c9302781e69c453f8e/dubbo-plugin/dubbo-qos/src/main/java/com/alibaba/dubbo/qos/command/impl/Online.java) 和 [`com.alibaba.dubbo.qos.command.impl.Online`](https://github.com/YunaiV/dubbo/blob/703764e6e82a72ddca7401c9302781e69c453f8e/dubbo-plugin/dubbo-qos/src/main/java/com/alibaba/dubbo/qos/command/impl/Online.java) 类。

## 5.2 ConsumerInvokerWrapper

[`com.alibaba.dubbo.registry.support.ConsumerInvokerWrapper`](https://github.com/YunaiV/dubbo/blob/703764e6e82a72ddca7401c9302781e69c453f8e/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/support/ConsumerInvokerWrapper.java) ，实现 Invoker 接口，服务消费者 Invoker Wrapper ，代码如下：

```
/**
 * Invoker 对象
 */
private Invoker<T> invoker;
/**
 * 原始 URL
 */
private URL originUrl;
/**
 * 注册中心 URL
 */
private URL registryUrl;
/**
 * 消费者 URL
 */
private URL consumerUrl;
/**
 * 注册中心 Directory
 */
private RegistryDirectory registryDirectory;
```

- 相比纯粹的 Invoker 对象，又多了运维命令需要的属性。例如 `registryDirectory` 属性，可以在使用**列出消费者和提供者命令**后，输出可消费者**可调用**的服务提供者数量 。想提前深入了解的胖友，可以看下 [`com.alibaba.dubbo.qos.command.impl.Ls`](https://github.com/YunaiV/dubbo/blob/703764e6e82a72ddca7401c9302781e69c453f8e/dubbo-plugin/dubbo-qos/src/main/java/com/alibaba/dubbo/qos/command/impl/Ls.java) 类。

# 5. integration

不同于上面我们看到的代码，[`integration`](https://github.com/YunaiV/dubbo/tree/414a4799cef08f0b5263d838eeaf8d8f169f2cdc/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/integration) 包下是对其他 Dubbo 模块的集成：

- RegistryProtocol ，对 `dubbo-rpc-api` 的依赖集成。
- RegistryDirectory ，对 `dubbo-cluster` 的依赖集成。

考虑到超出了本文的范畴，后面涉及到时，单独分享。

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

对注册中心的使用，不熟悉的胖友，可能理解起来会有点懵。嘿嘿，仿佛为自己写的差，找了一个理由。哈哈哈。

下一篇，我们来结合 Zookeeper ，进一步理解。

另外，胖友也可以多多调试噢。