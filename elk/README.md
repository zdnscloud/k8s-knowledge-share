# Kubernetes集群中日志收集方案-EFK
- Elasticsearch 是一个实时的、分布式的可扩展的搜索引擎，允许进行全文、结构化搜索，它通常用于索引和搜索大量日志数据，也可用于搜索许多不同类型的文档
- Kibana 是 Elasticsearch 的一个功能强大的数据可视化 Dashboard，Kibana 允许你通过 web 界面来浏览 Elasticsearch 日志数据
- Fluentd是一个流行的开源数据收集器，我们将在 Kubernetes 集群节点上安装 Fluentd，通过获取容器日志文件、过滤和转换日志数据，然后将数据传递到 Elasticsearch 集群，在该集群中对其进行索引和存储
- fluentd bit是纯C写的，功能与fluentd类似，与k8s的结合更深 https://github.com/fluent/fluent-bit-docs

![""](fluent-bitVSfluentd.png)

## fluent-bit流程
![""](fluent-bit.jpg)

