# prometheus-operator-架构和搭建
### 作者:流火夏梦


**普罗米修斯架构图：**
![](https://i.imgur.com/1buyJ1g.png)

** 功能组件解析


** 1. Prometheus Server

Prometheus Server 负责从 Exporter 拉取和存储监控数据，并提供一套灵活的查询语言（PromQL）供用户使用。



** 2. Exporter

Exporter 负责收集目标对象（host, container…）的性能数据，并通过 HTTP 接口供 Prometheus Server 获取。

** 

** 3. 可视化组件

监控数据的可视化展现对于监控方案至关重要。以前 Prometheus 自己开发了一套工具，不过后来废弃了，因为开源社区出现了更为优秀的产品 Grafana。Grafana 能够与 Prometheus 无缝集成，提供完美的数据展示能力。


** 4. Alertmanager**

用户可以定义基于监控数据的告警规则，规则会触发告警。一旦 Alermanager 收到告警，会通过预定义的方式发出告警通知。支持的方式包括 Email、PagerDuty、Webhook 等.

其他监控方案参考
 Zabbix、Graphite、Nagios 

### Prometheus 的解决方案。
Prometheus 只需要定义一个全局的指标 container_memory_usage_bytes，然后通过添加不同的维度数据来满足不同的业务需求。

比如对于前面 webapp1 的三条取样数据，转换成 Prometheus 多维数据将变成：
![](https://i.imgur.com/oIrySZQ.png)



#### Prometheu数据模型分析:
后面三列 container_name、image、env 就是数据的三个维度。
想象一下，如果不同 env（prod、test、dev），不同 image（mycom/webapp:1.2、mycom/webapp:1.3）的容器，它们的内存使用数据中标注了这三个维度信息，那么将能满足很多业务需求，比如：
计算 webapp2 的平均内存使用情况：avg(container_memory_usage_bytes{container_name=“webapp2”})
计算运行 mycom/webapp:1.3 镜像的所有容器内存使用总量：sum(container_memory_usage_bytes{image=“mycom/webapp:1.3”})

#### Prometheus 数据模型的优势：
通过维度对数据进行说明，附加更多的业务信息，进而满足不同业务的需求。同时维度是可以动态添加的，比如再给数据加上一个 user 维度，就可以按用户来统计容器内存使用量了。Prometheus 丰富的查询语言能够灵活、充分地挖掘数据的价值。前面示例中的 avg、sum、by 只是查询语言中很小的一部分功能，已经为我们展现了 Prometheus 对多维数据进行分片、聚合的强大能力。

## Prometheus 容器集群快速搭建：（基于docker分布式容器进行部署方案）

### ** 组件部署分析:**

Prometheus Server   整个监控杆系统大脑

Grafana   系统数据展示

cAdvisor   系统数据采集




##### Prometheus Server
Prometheus Server 本身也将以容器的方式运行在master 上



#####  Exporter
我们将使用：Node Exporter，负责收集 host 硬件和操作系统数据。它将以容器方式运行在所有 host 上。

##### cAdvisor
cAdvisor负责收集容器数据。(它将以容器方式运行在所有 host 上)

##### Grafana
显示多维数据，Grafana 本身也将以容器方式运行在 host 192.168.56.103 上.

**步骤1:**

运行 Node Exporter(每一个监控节点都需要运行以下命令)

docker run -d -p 9100:9100 \
-v "/proc:/host/proc" \
-v "/sys:/host/sys" \
-v "/:/rootfs" \
prom/node-exporter \
--path.procfs /host/proc \
--path.sysfs /host/sys \
--collector.filesystem.ignored-mount-points "^/(sys|proc|dev|host|etc)($|/)"

--net=host \     参数可选

验证:
通过访问Node Exporter:9100    进行数据验证




**步骤2:**
运行 cAdvisor

docker run \
--volume=/:/rootfs:ro \
--volume=/var/run:/var/run:rw \
--volume=/sys:/sys:ro \
--volume=/var/lib/docker/:/var/lib/docker:ro \
--publish=8080:8080 \
--detach=true \
--name=cadvisor \
google/cadvisor:版本号


--net=host \     参数可选




验证:
通过访问Node Exporter:8080    进行数据验证


##### **步骤3:**

运行 Prometheus Server（主节点运行普罗米修斯）

docker run -d -p 9090:9090 \
-v /root/prometheus.yml:/etc/prometheus/prometheus.yml \
--name prometheus \
--net=host \
prom/prometheus


--net=host \     参数可选
这样 Prometheus Server 可以直接与 Exporter 和 Grafana 通信。
prometheus.yml 是 Prometheus Server 的配置文件，需要作出的修改是

最重要的配置是：
static_configs:
-targets:['localhost:9090','localhost:8080','localhost:9100','192.168.56.102:8080','192.168.56.102:9100']


验证:需要通过访问master:9090  查看返回的数据和界面







**步骤4:**


运行 Grafana（主节点运行普罗米修斯）

docker run -d -i -p 3000:3000 \
-e "GF_SERVER_ROOT_URL=http://grafana.server.name" \
-e "GF_SECURITY_ADMIN_PASSWORD=secret" \
grafana/grafana

--net=host \     参数可选


验证:需要通过访问master:3000  配置界面



### **容器云平台监控工具方案的比较:**
1. 		ps/top/stats
1. 		Sysdig
1. 		Weave Scope
1. 		cAdvisor
1.      Prometheus

![](https://i.imgur.com/S5yIZSh.png)


###### **选择普罗米修斯的原因:**
Prometheus 的数据模型和架构决定了它几乎具有无限的可能性。Prometheus 和 Weave Scope 都是优秀的容器监控方案。除此之外，Prometheus 还可以监控其他应用和系统，更为综合和全面。

### **K8s集群监控方案选型:**

- 1 Weave Scope
- 2 Heapster 是 Kubernetes 原生的集群监控方案
- 3 Prometheus Operator监控方案
- Prometheus Operator 是 CoreOS 开发的基于 Prometheus 的 Kubernetes 监控方案，是目前功能最全面的开源方案


## **Prometheus Operator 架构**
![](https://i.imgur.com/WYMqRwl.png)


## Operator ##

Operator 即 Prometheus Operator，在 Kubernetes 中以 Deployment 运行。其职责是部署和管理 Prometheus Server，根据 ServiceMonitor 动态更新 Prometheus Server 的监控对象。

## Prometheus Server ##

Prometheus Server 会作为 Kubernetes 应用部署到集群中。为了更好地在 Kubernetes 中管理 Prometheus，CoreOS 的开发人员专门定义了一个命名为 Prometheus 类型的 Kubernetes 定制化资源。我们可以把 Prometheus看作是一种特殊的 Deployment，它的用途就是专门部署 Prometheus Server。

## Service ##

这里的 Service 就是 Cluster 中的 Service 资源，也是 Prometheus 要监控的对象，在 Prometheus 中叫做 Target。每个监控对象都有一个对应的 Service。比如要监控 Kubernetes Scheduler，就得有一个与 Scheduler 对应的 Service。当然，Kubernetes 集群默认是没有这个 Service 的，Prometheus Operator 会负责创建。

## ServiceMonitor ##

Operator 能够动态更新 Prometheus 的 Target 列表，ServiceMonitor 就是 Target 的抽象。比如想监控 Kubernetes Scheduler，用户可以创建一个与 Scheduler Service 相映射的 ServiceMonitor 对象。Operator 则会发现这个新的 ServiceMonitor，并将 Scheduler 的 Target 添加到 Prometheus 的监控列表中。

ServiceMonitor 也是 Prometheus Operator 专门开发的一种 Kubernetes 定制化资源类型。

## Alertmanager ##

除了 Prometheus 和 ServiceMonitor，Alertmanager 是 Operator 开发的第三种 Kubernetes 定制化资源。我们可以把 Alertmanager 看作是一种特殊的 Deployment，它的用途就是专门部署 Alertmanager 组件。




# 普罗米修斯全局架构: 
![](https://i.imgur.com/l4GJCFJ.png)

## prometheus-operator快速部署: ##


### 使用prometheus监控k8s一般有三种方案: 
- 官方helm安装
- https://github.com/giantswarm/kubernetes-prometheus
- https://github.com/coreos/prometheus-operator/releases?after=v0.13.0


安装要求:系统k8s集群至少达到v1.7.0以上才行。这里是基于k8s1.9.2进行的部署。

##### 由于国内网络环境问题不建议使用helm ##
**其中第二种方案比较推荐,coreos公司维护, 更新非常快, 但一定要注意使用的版本号. 这里只介绍第二种方案；**

## 下载 ##
###### 拉取源代码
git clone https://github.com/coreos/prometheus-operator.git
cd prometheus-operator
###### 切换tag
git checkout v0.19.0



[prometheus-operator分支对照](https://github.com/coreos/prometheus-operator/releases "分支版本")

## **安装**
###### 创建命名空间
kubectl create namespace monitoring
###### 创建prometheus-operator
kubectl apply -f bundle.yaml -n monitoring
###### 创建相关服务
- cd contrib/kube-prometheus
- ./hack/cluster-monitoring/deploy

![](https://i.imgur.com/VZDOR5e.png)


![](https://i.imgur.com/7oZ9e3u.png)



#### 根据开放端口进行服务映射访问： ##
1. prometheus-opeated     Master:31826
1. grafana				   Master:32450
1. altermanager		   Master:32603


![](https://i.imgur.com/fGH2tDJ.png)

**如果发现存在kubelet没有起来，则需要检查rbac是否进行验证，或者开启全局访问k8sapi权限。**

但是为了安全问题，最好还是自己麻烦点配置一下rbac认证吧!

**访问告警页面进行配置**

![](https://i.imgur.com/xolI4ql.png)

**访问数据可视化页面进行配置**
![](https://i.imgur.com/bZqccIN.png)




**如果在自己的k8s集群种发现数据已经可以在前端进行数据展示，证明自己搭建监控集群系统已经成功！**
