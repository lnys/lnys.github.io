---
title: 基于Kubernetes（k8s）的RabbitMQ 集群
---
目前，有很多种基于Kubernetes搭建RabbitMQ集群的解决方案。今天笔者今天将要讨论我们在Fuel CCP项目当中所采用的方式。这种方式加以转变也适用于搭建RabbitMQ集群的一般方法。所以如果你想要设计自己的解决方案，你应该收集一些更符合你定制化需求的文章。

命名你的集群

在Kubernetes内部运行RabbitMQ集群会遇到一系列有意思的问题。最先会遇到的问题是为了使各个节点之间互相可见，我们应该如何命名各个节点。以下是一些符合规范的不同的命名方法:

```rabbit@hostname
rabbit@hostname.domainname
rabbit@172.17.0.4```
在你尝试着启动第一个节点之前，你需要确定容器之间可以通过选取名字的方式互通。例如，Ping命令可以访问@符号后面的节点名称。

Erlang分布式方案(RabbitMQ所基于的实现方式)可以运行在两种命名方案当中的一种：短节点名或者长节点名。区别的关键点在于：名字中如果存在“.”，就属于长节点名；否则就是短节点名。在以上节点名举例当中，第一个就属于短节点名；第二个和第三个则属于长节点名。

综合以上要求，我们可以有以下节点命名规则以供选择:

使用Pet集合（或者叫有状态集合）：我们可以使用相对固定的DNS名字。与一般可以“丢弃”的副本出故障时可以轻易的踢出集群相对应，Pet集合是一组有状态的Pod，每个Pod有着可以表示其状态的标识符。
使用IP地址以及一些具有自动发现集群节点的工具（例如，autocluster插件可以使RabbitMQ节点自动发现集群子节点）。
以上的命名规则均需要采用长名字模式。但是DNS/节点名在K8S Pod内部配置的方式需要RabbitMQ 3.6.6以后版本才可以支持。所以如果你采用这种方式，请确认你的RabbitMQ的版本。

Erlang的Cookie问题

第二个成功的关键问题是RabbitMQ节点需要共享一个密钥Cookie。默认情况下RabbitMQ从一个文件当中读取这个Cookie(如果该文件不在，则自动生成一个)。为确保该Cookie在所有节点一致，我们有以下方案:

在制作Docker镜像的时候生成该Cookie文件。这种方法并不推荐，因为该Cookie可以让你对整个RabbitMQ内部有完整的访问权限。
用一个Entrypoint脚本文件来生成该Cookie文件。生成时可以用环境变量来传输密文。如果我们还需要Entrypoint脚本文件来做其他事情，这个方案和下一个方案都可以使用。
通过环境变量的方式给RabbitMQ传递:
RABBITMQ_CTL_ERL_ARGS=”-setcookie ”
RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS=”-setcookie ”
集群的缺陷

于RabbitMQ集群另外一个必须知道是事情是当一个节点加入集群时，该节点的数据将会丢失，无论是什么样的数据。在常见的用例当中，这点可能无关紧要。例如，当一个空节点加入到集群当中的时候，这时候该节点不存在任何数据丢失的问题。但是，当我们有两个节点，他们已经相对独立的运行过一段时间之后，并且已经累积了一些数据时。这时候我们无法在不损失任何数据的前提下将他们合并（注意：在网络中断或者节点宕机后回复一个集群都会有同样的问题，同样会有数据丢失）。对于特殊的数据，你可以使用一些其他的解决方案。例如，把需要重置的节点中的数据先备份出来。但是，目前没有一个健壮的、自动的、全局的解决方案。

所以我们的自动化集群解决方案受到我们能容忍什么样的数据丢失这一方面的影响。

Cluster formation集群信息

假设你已经解决了所有命名规则相关的问题，并且用rabbitmqctl命令手工的建立了集群。现在是把我们的RabbitMQ封装成自动化集群的时候了。就像我们以前遇到的问题一样，没有一个解决方案可以解决所有的问题。

一个特定的解决方案只适合我们并不关心服务停止或者链接丢失时数据丢失的问题。这种场景的一个例子就是当你使用RPC发送请求时，客户端可以在超时或者收到错误时重发请求。当服务恢复之后，RPC请求将不再有效，相关数据也不再有任何意义。幸运的是各个OpenStack组件之间的RPC调用正是这种情况。

在思考过以上所有问题之后，现在我们可以搭建我们自己的解决方案了。无状态是我们的首选，所以我们选择了IP地址而不是Pet集合。Autocluster插件将是我们组织动态节点集群的首选。

查看过Autocluster的文档之后，我们得出了以下配置项:

{backend, etcd}: 这几乎是我们唯一的选择。Consul或者K8S可以很好的工作。选择它的唯一理由也是因为测试起来很简单。你可以下载etcd的二进制版本，不需要任何参数就可以运行，这样可以搭建起一个基于本地的集群。
{autocluster_failure, stop}:如果一个pod无法加入集群对于我们来说就没有任何价值。这个pod将会被移除集群，并在一个相对安全的环境中重启。
{cluster_cleanup, true}, {cleanup_interval, 30},{cleanup_warn_only, false}, {etcd_ttl, 15}:一个节点只有在完全启动，并且成功加入集群之后才可以在etcd中注册。只要该节点还可用，注册用的TTL将会不间断的被更新。如果该节点停掉（或者由于某种原因无法更新TTL）,他将会被强制从集群中删除。即使该节点重启后获得了相同的IP地址，它也会被认为是重新加入集群。
无法预料的竞争

如果你尝试着按照以上搭建了几次集群，你会发现有时整个集群会分裂成几个互相无法联通的小集群。问题产生的原因是启动时对于竞争问题的唯一保护措施在节点启动时会有随机的延迟现象。在最坏的情况下，每个节点会认为它是第一个节点（例如这时在etcd中没有任何记录），所以这个节点就以无集群模式启动了。

Autocluster最终为该问题提出了一个很大的补丁。它在启动的过程中增加了锁机制：节点在启动时最先申请启动锁资源，最后在该节点成功在后台注册之后才释放。目前只有etcd这个平台支持该功能。但是其它平台也可以很容易的支持该功能(在后台当中增加两个新的回调函数)。

另一个问题是：K8S、Pet集合他们可以在启动的时候进行编排工作；在任何时间都只有一个节点处于启动状态。 该功能只处于初期测试阶段。提供该功能的补丁不仅仅提供给K8S的用户，而是提供给所有平台的开发者。

监控

目前唯一遗留的问题就是为我们运行在非看管状态的集群增加一个监控系统。我们需要监控rabbitmq的健康状况以及它是否和集群其它节点良好的工作在一起。

你也许还记得以前可以通过运行rabbitmqctl list_queues或者rabbitmqctl list_channels来监控。但是这种方案并不完美，因为它无法区别本地和远程的问题。同时它又明显的增加了网络负载。为了弥补这个缺陷，3.6.4版本之后推出了新的、轻量级的rabbitmqctl node_health_check。这是检查rabbitmq集群当中单节点健康状况最好的方法。

检查一个节点是否正确的加入集群需要做几方面的检查:

新加入的节点应当与Autocluster后台最优节点在一起注册成集群。这个最优节点是在注册成功列表里按字母顺序排在最前面的节点。
即使当节点已经与现有节点形成了集群，但它的数据可能还是分开的。对于这个检查并不是分开的，我们需要在当前节点和新发现节点都检查分区列表。
所有的这些可以通过独立的检查完成，检查可以使用以下命令:
rabbitmqctl eval ‘autocluster:cluster_health_check_report().’

使用rabbitmqctl命令我们可以检测出rabbitmq节点的任何问题，也可以通过该命令将该节点停掉。因此，K8S可以有机会施展魔法重启该节点。

搭建你自己的RabbitMQ集群

如果你自己亲自按照这个方案搭建这个集群，你需要最新版本的RabbitMQ以及autocluster插件的定制化版本（目前启动锁机制这个补丁还没有合并到主版本当中）。

你可以查看Fuel CCP如何搭建集群，或者用你自己的实现方法使用独立的版本来搭建。

为了提示你该方案案如何实施，让我们假设你已经复制了第二个repository，并且你已经有了一个叫做“demo”的K8S的命名空间。Etcd的服务已经在K8S集群中运行，并且可以通过”etcd”这个名字访问。你可以通过以下命令来搭建环境:

kubectl create namespace demo
kubectl run etcd --image=microbox/etcd --port=4001 \
--namespace=demo -- --name etcd
kubectl --namespace=demo expose deployment etcd
完成以上步骤后，用以下命令搭建RabbitMQ:

1. 用合适的RabbitMQ和Autocluster版本创建Docker镜像，并设置相应配置文件。

$ docker build . -t rabbitmq-autocluster
2. 保存Erlang的Cookie到K8S当中。

$ kubectl create secret generic --namespace=demo erlang.cookie \
--from-file=./erlang.cookie
3. 创建3节点RabbitMQ集群。为了简化操作，你们可以从以下链接下载rabbitmq.yaml文件https://github.com/binarin/rabbit-on-k8s-standalone/blob/master/rabbitmq.yaml.

$ kubectl create -f rabbitmq.yaml
4. 检查集群是否正常工作
```
$ FIRST_POD=$(kubectl get pods --namespace demo -l 'app=rabbitmq' \
-o jsonpath='{.items[0].metadata.name }')
$ kubectl exec --namespace=demo $FIRST_POD rabbitmqctl \
cluster_status
```
你应该得到如下输出:
```
Cluster status of node 'rabbit@172.17.0.3' ...
[{nodes,[{disc,['rabbit@172.17.0.3','rabbit@172.17.0.4',
​                'rabbit@172.17.0.7']}]},
 {running_nodes,['rabbit@172.17.0.4','rabbit@172.17.0.7','rabbit@172.17.0.3']},
 {cluster_name,<<"rabbit@rabbitmq-deployment-861116474-cmshz">>},
 {partitions,[]},
 {alarms,[{'rabbit@172.17.0.4',[]},
​          {'rabbit@172.17.0.7',[]},
​          {'rabbit@172.17.0.3',[]}]}]

```
这里最关键的一点是nodes与running nodes集合均有三个节点。

```