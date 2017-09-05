# EFK in Kubernetes
##  1. 概述

企业中常用的日志系统莫过于ELK，即Elasticsearch（简称ES）、Logstash和Kibana。
- Logstash：负责收集日志
- Elasticsearch：负责日志的存储和检索
- Kibana：向用户提供查询和显示的操作界面。

由于Logstash是用JRuby开发的，需要JVM才能运行，作为日志采集器显得有些庞大。因此我们把Logstash换成了Fluentd。Fluentd是用ruby开发的，内存开销只有Logstash的十分之一，而且有很多插件，方便与其它工具集成。最关键的Fluentd有一个**kubernetes_metadata_filter**的插件可以在原始的日志内容上附加kubernetes的信息。

本文介绍了在多个kubernetes集群中收集日志的方案，涉及到日志的采集，传送，存储，认证和授权等方面的内容。

##  2. 总体架构

如下图所示，日志系统由两部分组成：**日志采集**和**日志分析**。
- 日志采集：运行于Kubernetes集群内部，通过DaemonSet的方式收集每个Node上的日志。
- 日志分析：可以独立于Kubernetes集群之外，通过Kafka收集Fluentd采集的各Node上的日志。

目前，我们的架构中，有多个Kubernetes集群，各集群的日志都发到同一个日志分析模块。
![架构图](./arch.png)

##  3. 日志采集
### a. Docker日志驱动的选择

日志采集通过Fluentd从Docker容器中收集日志。Docker有几种日志驱动可以用于Fluentd收集日志：
- json-file：容器日志以json格式写进文件，fluentd用tail收集。
- syslog：fluentd以syslog方式收集，容器日志发送给fluentd打开的syslog监听端口。
- journald：容器日志写入journald，fluentd安装systemd插件，收集journald的日志。
- fluentd：容器日志直接发送给fluentd，fluentd以forward方式打开监听，接收容器日志。

我们不但要收集容器的日志，还要使用fluentd的kubernetes_metadata_filter插件在日志上附加上Kubernetes的元数据。kubernetes_metadata_filter只支持journald和json-file两种驱动，所以我们的选择范围也就集中到这两种方式上。另外，docker logs命令也只支持json-file和journald两种试，如果配置成其它驱动方式，将不能使用docker logs命令。

最终我们选择了json-file驱动，这也是Docker的默认驱动，我们比较熟悉。json-file还可以通过log-opts设置每个容器的日志文件的大小和数量，方便预留日志存储空间。fluentd自带的tail模块就可以收集日志文件，如果用journald，还需要为fluentd额外安装插件。

### b. Kubernetes日志与Docker日志的关系

如果Docker配置的是json-file日志驱动，那么kubelet在启动容器时，会同时创建一个符号链接到Docker容器的日志文件。
kubernetes的日志文件默认存储在 **/var/log/containers/** 目录下，以<pod名>_<namespace>_<container名>_<container Id>.log命名。
这个文件是一个符号链接，链接到Docker容器的日志文件。Docker日志文件默认在 **/var/lib/docker/containers/<container Id>/** 目录下，以<container Id>-json.log命名。fluentd可以用tail模块监视/var/log/containers/目录，收集此目录下的文件内容。

### c. kuberntetes_metadata_filter插件的工作原理

fluentd的kubernetes_metadata_filter插件通过分析/var/log/conatiners/目录下日志文件的名称，提取出namespace和Pod名称。然后插件再调用Kubernetes的API Server的接口，查询出Pod的其它信息，再将查询出的Pod元数据附加到每条日志上。为了减少对API Server的查询频率，插件缓存了Kubernetes的Pod信息，不需要每条日志都向API Server查询。插件向日志中增加的Pod的元数据包括：namespace名称、pod的Id、pod的名称、pod的labels、pod所在的host（nodeName)、master_url和pod的annotations。

### d.  Fluentd的部署

Fluentd通过Kubernetes的DaemonSet部署，在每个Node节点启动一个Fluentd Pod收集本节点的Pod日志。下面的Yaml文件供大家参考。有几点说明：
- 因为我们的Kubernetes集群启用了RBAC，所以要为Fluentd配置权限。我们创建了名为fluentd的ServiceAccount、ClusterRole和ClusterRoleBinding。
- 目前Fluentd的配置存储在ConfigMap中，通过Volume挂载入容器，将来计划将固定的配置直接写入fluentd的镜像。
- Fluentd需要把主机的/var/lib/docker/containers和/var/log两个目录挂载入容器中。
- 关于Fluentd镜像，现阶段只是在官方v0.12版本镜像的基础上增加了所需的插件，并且以root用户运行，因为/var/lib/docker/containters下的内容只能是root用户才能读取。
- Fluentd并不需要容器网络，所以使用了hostNetwork
- 我们额外配置了kubernetes_url，没有用Kubernetes注入的kubernetes服务的连接信息，这样可以不依赖kube-proxy。
- kubernetes_url使用了本地域名k8s.local，我们在每个node的/etc/hosts中配置了k8s.local的IP地址，由于我们使用提hostNetwork在容器中可以加载到node的/etc/hosts内容。
