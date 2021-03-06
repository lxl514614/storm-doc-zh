---
title: Storm Metrics
layout: documentation
documentation: true
---
Storm 开放了一个 metrics 接口，用来汇报整个 topology 的汇总信息 。
Storm 内部通过该接口可以跟踪各类统计数据：executor 和 acker 的数量；每个 bolt 的平均处理时延、worker 节点的堆栈使用情况，这些信息都可以在 Nimbus 的 UI 界面中看到。

### Metric Types

Metrics 必须实现  [`IMetric`]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/metric/api/IMetric.java)接口,IMetric接口只包含一个方法  `getValueAndReset` -- 得到汇总值，并且重置为初始状态。例如，在 MeanReducer 中实现 running total/running count 的均值，然后两个值都重新设置为0.

Storm 提供了以下几种 metric 类型：

* [AssignableMetric]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/metric/api/AssignableMetric.java) -- 将 metric 设置为指定值。此类型在两种情况下有用：1. metric 本身为外部设置的值；2. 你已经另外计算出了汇总的统计值。
* [CombinedMetric]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/metric/api/CombinedMetric.java) -- 可以对 metric 进行关联更新的通用接口。
* [CountMetric]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/metric/api/CountMetric.java) -- 返回 metric 的汇总结果。可以调用 `incr()` 方法来将加过自增；调用 `incrBy(n)` 方法来将结果加上或者减去给定值。
  - [MultiCountMetric]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/metric/api/MultiCountMetric.java) -- 返回包含一组 CountMetric 的 HashMap
* [ReducedMetric]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/metric/api/ReducedMetric.java)
  - [MeanReducer]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/metric/api/MeanReducer.java) -- 跟踪 `reduce()` 方法提供的运行状态均值结果（可以接受 `Double`、`Integer`、`Long` 等类型，内置的均值结果是 `Double` 类型）。MeanReducer 确实是一个相当棒的家伙。
  - [MultiReducedMetric]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/metric/api/MultiReducedMetric.java) -- 返回包含一组 ReducedMetric 的 HashMap


### Metrics Consumer

你可以监听和处理 topology（拓扑）的metrics ，通过 注册 Metrics Consumer到你的topology.

注册 metrics consumer 到你的 topology，添加到你的 topology（拓扑）配置，像下面这样：

```java
conf.registerMetricsConsumer(org.apache.storm.metric.LoggingMetricsConsumer.class, 1);
```

你可以参考 [Config#registerMetricsConsumer](javadocs/org/apache/storm/Config.html#registerMetricsConsumer-java.lang.Class-) ，然后根据javadoc 覆盖 java 方法.

否则编辑 storm.yaml 配置文件：

```yaml
topology.metrics.consumer.register:
  - class: "org.apache.storm.metric.LoggingMetricsConsumer"
    parallelism.hint: 1
  - class: "org.apache.storm.metric.HttpForwardingMetricsConsumer"
    parallelism.hint: 1
    argument: "http://example.com:8080/metrics/my-topology/"
```

注册 metrics consumer后，Storm 会添加 MetricsConsumerBolt 到你的 topology，每个 MetricsConsumerBolt 会从所有的 tasks 接受 metrics。 相应的 MetricsConsumerBolt 的 parallelism 设置为 `parallelism.hint` ，`component id` 设置为 `__metrics_<metrics consumer class name>`. 如果你注册多次相同的类， component id 后会添加 `#<sequence number> `。

Storm 提供了一些内置的 metrics consumers，我们来看一下 topology（拓扑）中提供了哪些 metrics.

* [`LoggingMetricsConsumer`]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/metric/LoggingMetricsConsumer.java) -- 监听所有的 metrics ，然后将数据dump 到日志文件(Tab Separated Values).
* [`HttpForwardingMetricsConsumer`]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/metric/HttpForwardingMetricsConsumer.java) -- 监听所有的 metrics，并且将数据序列化，然后通过http post 到配置的url。Storm 提供  [`HttpForwardingMetricsServer`]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/metric/HttpForwardingMetricsServer.java) 抽象类，你可以继承这个类，并且启动一个 HTTP sever，通过 HttpForwardingMetricsConsumer 处理发送的 metrics.

当然，Storm 开放了实现 Metrics Consumer 的  [`IMetricsConsumer`]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/metric/api/IMetricsConsumer.java) 接口，你可以创建自定义 metrics consumers ，绑定到相应的 topology（拓扑）。或者使用 Storm 社区内其他比较好的 Metrics Consumers 实现. 你可以参考 [versign/storm-graphite](https://github.com/verisign/storm-graphite) 和 [storm-metrics-statsd](https://github.com/endgameinc/storm-metrics-statsd).

当你实现自己的 metrics consumers，调用 [IMetricsConsumer#prepare](javadocs/org/apache/storm/metric/api/IMetricsConsumer.html#prepare-java.util.Map-java.lang.Object-org.apache.storm.task.TopologyContext-org.apache.storm.task.IErrorReporter-)的时候，`argument` 需要传给 consumer 对象。所以你要参考 yaml 文件中配置的Java 类型，做好类型的分配。

请记住 MetricsConsumerBolt 只是 Bolt 类型的一种，所以 如果 metrics consumers 如果不能持续处理 metrics，topology 的吞吐量将会下降，所以你要和其他 Bolt 一样关注好 MetricsConsumerBolt。一个比较好的方式就是将 Metrics Consumers 设计为 `non-blocking`（非阻塞）的。

### 构建你自己的 metric（task level）

你可以通过注册 `IMetric` 到 Metric Registry（登记处），然后度量你的 metric. 
假定你想要度量 Bolt#execute 的执行次数。我们一起来定义这个 metric 实例。CountMetric 符合我们的应用场景。

```java
private transient CountMetric countMetric;
```

你会发现，我们将CountMetric 定义为 transient。因为IMetric 并不是 Serializable 的，所以定义为 transient 可以避免很多问题。

下一步，我们初始化和注册 metric 实例。

```java
@Override
public void prepare(Map conf, TopologyContext context, OutputCollector collector) {
	// other intialization here.
	countMetric = new CountMetric();
	context.registerMetric("execute_count", countMetric, 60);
}
```

第一个和第二个参数很简单，metric 名称和 IMetric 实例。第三个参数[TopologyContext#registerMetric ](javadocs/org/apache/storm/task/TopologyContext.html#registerMetric-java.lang.String-T-int-)是发布和重置 metric 的间隔时间。

最后，当 Bolt.execute 执行的时候，自增countMetric的值。

```java
public void execute(Tuple input) {
	countMetric.incr();
	// handle tuple here.	
}
```

请注意， topology 的sample rate 不适用于自定义 metrics，所以我们自己调用 incr() 方法。

`countMetric.getValueAndReset()` 每隔60秒都会被调用， ("execute_count", value)也会被发送到 MetricsConsumer。


### Build your own metrics (worker level)

你可以注册 worker 级别的 metrics，将他们添加到集群所有的workers 的 `Config.WORKER_METRICS` 配置，或者所有workers上的指定 topology，通过 `Config.TOPOLOGY_WORKER_METRICS` 配置。

例如，我们可以添加 `worker.metrics` 配置到集群的 storm.yaml。

```yaml
worker.metrics: 
  metricA: "aaa.bbb.ccc.ddd.MetricA"
  metricB: "aaa.bbb.ccc.ddd.MetricB"
  ...
```

或者按照 `Map<String,String>`（metric name, metric class name）格式，key是`Config.TOPOLOGY_WORKER_METRICS`，来设置 config map。

worker metrics 实例有下面一些限制：

A) worker 级别的 metrics 应该是一种 gauge 类，因为它是从 SystemBolt 初始化和注册的，不会暴露给 user tasks。

B) Metrics 通过默认的构造器初始化，并且不会对执行配置注入或者对象注入。

C) metrics Bucket size(secounds) 已经修正为 `Config.TOPOLOGY_BUILTIN_METRICS_BUCKET_SIZE_SECS`.


### Builtin Metrics


Storm 的  [builtin metrics]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/daemon/builtin_metrics.clj) 工具。

[builtin_metrics.clj]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/daemon/builtin_metrics.clj) 为内置的metrics 设置了数据结构，其他框架组件可以使用 facade 方法来更新数据。metrics在被调用的时候走计算逻辑-- 可以看例子 [`ack-spout-msg`]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/daemon/executor.clj#358)  的 `clj/b/s/daemon/daemon/executor.clj` 部分。

