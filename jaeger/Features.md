### Features

Jaeger用于微服务监控和故障排除的分布式跟踪系统，它的主要特性有：
1. 分布式上下文传播
2. 分布式事务监控
3. 根本原因分析
4. 服务依赖分析
5. 性能/延迟优化

#### 可扩展性
Jaeger后端设计的两个特性：
1. 没有单点故障
2. 根据业务需要动态扩容

例如：在Uber任意给定的一个Jaeger安装可以很容易地每天处理几十亿spans

#### 对OpenTracing的原生支持
Jaeger后端、Web UI和工具库都被设计成从底层就支持OpenTracing标准
1. 通过span的引用，traces是一个有向无环图
2. 支持强大地带自定义类型的tag和结构化日志
3. 通过baggage，支持通用分布式上下文传播机制

#### 多后端存储支持
Jaeger支持两种流行的开源NoSQLs数据库，作为trace存储：Cassandra 3.4+ 和 ElasticSearch 5.x/6.x. 同时Jaeger也正在和其他数据库展开合作，包括：ScyllaDB, InfluxDB, Amazon DynamoDB. 为了支持简单的测试使用，Jaeger也可用于内存作为后端存储，这仅仅是用于测试使用。
#### 现代化的Web UI
Jaeger WebUI 是使用React开发实现的。几个优化点在v1.0版本发布，主要是允许UI有效地处理大量数据，并且展示带上千个spans的traces(例如：我们尝试了一个带有8000spans的trance)

#### 云原生部署
Jaeger后端是有多个Docker Images构成。这个二进制支持多种配置方法， 包括命令行参数，环境变量和多格式的配置文件(yaml, toml等等)， 支持使用K8s模板和Helm chart来部署k8s集群

#### 观测
所有Jaeger后端组件都暴露了Prometheus默认的metrics(其他后端metrics也被支持), 日志通过[zap](https://github.com/uber-go/zap)写入

#### 与ZipKin兼容性
尽管我们强烈推荐使用OpenTracing API检测应用程序，并且绑定到未来具有更多特性的Jaeger客户库, 但是如果你的组织已经投入使用了ZipKin库去检测应用程序，你不必要重写代码。Jaeger通过HTTP接收ZipKins格式(Thrift or JSON v1/v2)接收Zipkins定义的spans. 切换到Zipkin后端只是将来自ZipKin库的流量路由到Jaeger后端
