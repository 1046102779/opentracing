# Tracer目录

| trace | 系列文章 | 描述 |
| ------ | ------ | ------ |
| [Google Dapper](https://static.googleusercontent.com/media/research.google.com/en//archive/papers/dapper-2010-1.pdf) | [《分布式跟踪系统论文》](https://gocn.vip/article/849) | Dapper是分布式跟踪系统的研究论文，基本上厂商都是参考这篇论文设计的|
| 厂商trace产品对比 | [《分布式跟踪系统——产品对比》](https://gocn.vip/article/852) | 后续会陆续补充 |
| [OpenTracing标准](http://opentracing.io/) |  [OpenTracing——概念与目标](https://gocn.vip/article/854)|  |
| | [OpenTracing——相关概念术语](https://gocn.vip/article/856) |  |
| | [OpenTracing APIs](https://gocn.vip/article/858) |  |
| [opentracing-go](https://github.com/opentracing/opentracing-go) | [opentracing-go源码阅读一](https://gocn.vip/article/861) | 它是OpenTracing标准的schema实现|
|  | [opentracing-go源码阅读——信息携带](https://gocn.vip/article/862) |  |
|  | [opentracing-go源码阅读——Log存储(完结篇)](https://gocn.vip/article/864) | |
| [basictracer-go](https://github.com/opentracing/basictracer-go) | [basictracer源码阅读——TracerImpl](https://gocn.vip/article/868) | 它是对OpenTracing标准的最小实现，各大厂商可以不基于它实现自己的trace系统，直接以OpenTracing标准实现，并与basictracer-go同级 | 
|  | [basictracer-go源码阅读二——Span](https://gocn.vip/article/874) | |
|  | [basictracer-go源码阅读——event&propagation](https://gocn.vip/article/875) | |
|  | [basictracer-go源码阅读——SpanRecorder与wire](https://gocn.vip/article/876) | |
|  | [basictracer-go源码阅读——examples(完结)](https://gocn.vip/article/878) |  |
| [Appdash](https://github.com/sourcegraph/appdash) | [Appdash源码阅读——Tracer&Span](https://gocn.vip/article/879) | 如果从trace角度看Appdash，它并没有遵循OpenTracing，同时从如果不使用opentracing，则Appdash与OpenTracing标准没有任何关系。它的出生早于OpenTracing标准, 只不过后来对Appdash做了一个很小的扩展，而且设计考虑得很弱|
| | [Appdash源码阅读——Annotations与Event](https://gocn.vip/article/881) | |
| | [Appdash源码阅读——Recorder与Collector](https://gocn.vip/article/882) | |
| | [Appdash源码阅读——Store存储](https://gocn.vip/article/883) | |
| | [Appdash源码阅读——RecentStore和LimitStore](https://gocn.vip/article/892) | | 
| | [Appdash源码阅读——reflect](https://gocn.vip/article/893) | |
| | [Appdash源码阅读——部分opentracing支持](https://gocn.vip/article/894) | |
