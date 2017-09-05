# EFK in Kubernetes
##  1. 概述
   企业中常用的日志系统莫过于ELK，即Elasticsearch（简称ES）、Logstash和Kibana。Logstash负责收集日志，Elasticsearch负责日志的存储和检索，Kibana向用户提供查询和显示的操作界面。由于Logstash是用JRuby开发的，需要JVM才能运行，作为日志采集器显得有些庞大。因此我们把Logstash换成了Fluentd。Fluentd是用ruby开发的，内存开销只有Logstash的十分之一，而且有很多插件，方便与其它工具集成。最关键的Fluentd有一个kubernetes_metadata的插件可以在原始的日志内容上附加kubernetes的信息。
本文介绍了在多个kubernetes集群中收集日志的方案，涉及到日志的采集，传送，存储，认证和授权等方面的内容。
##  2. 总体架构
如下图所示，日志系统由两部分组成：日志采集和日志分析。日志采集部分运行于Kubernetes集群内部，通过DaemonSet的方式收集每个Node上的日志。日志分析部分可以独立于Kubernetes集群之外，通过Kafka收集Fluentd采集的各Node上的日志。目前，我们的架构中，有多个Kubernetes集群，各集群的日志都发到同一个日志分析模块。
![架构图](/path/to/img.jpg)
