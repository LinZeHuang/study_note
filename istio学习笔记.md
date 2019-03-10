## istio的介绍
Istio是由Google/IBM/Lyft共同开发的新一代Service Mesh开源项目。Istio 允许您连接、保护、控制和观测服务。


Istio 提供一种简单的方式来为已部署的服务建立网络，该网络具有负载均衡、服务间认证、监控等功能，只需要对服务的代码进行一点或不需要做任何改动。想要让服务支持 Istio，只需要在您的环境中部署一个特殊的 sidecar 代理，使用 Istio 控制平面功能配置和管理代理，拦截微服务之间的所有网络通信。


> *Q:如果是对kubernetes有了解的同学可能会想，在存在kube-proxy的情况下，因为kube-proxy也能把流量分发到各个pop中，配合其他组件也挺安全的，监控容器是否正常也有自己的策略，为何我们还要使用istio呢？*

确实kube-proxy能做到流量分发，但是他的分发策略只有两种，根据客户端ip固定分发到某个pop中，或者轮询，再更加复杂的场景就暂时还没有支持。如果要自己进行开发的话成本又很高。而istio对流量的控制策略多种多样。可以根据请求头部信息进行分发，小流量流入金丝雀版本，ab测试都可以支持。其他也是一样，在安全方面，对流量进行进行加密，在故障处理方面，可设置并发连接数，游服务请求数限制，在可以理解为Istio是对kubernetes已有流量功能的扩展。

> *Q:我们再和kube-proxy对比一下，kube-proxy是用iptable修改转发规则将流量导入kube-proxy，再由kube-proxy去负责分发，而istio却是用Envoy（C++ 开发的高性能代理）去作为代理？*

kube_proxy是部署在node上的，可能是在虚拟机上，可能是在实体机上，他们的ip映射规则是早就定好了。其实kube_proxy修改iptable来转发流量是迫不得已的，iptables模式问题不好定位，规则多了性能会显著下降，甚至会出现规则丢失的情况（kubernetes1.8开始新增了ipvs的支持）。

而istio是建立在pop上，它已经跳过了那个难受的iptable的阶段了，分发到istio上的流量都是“优质”的，确定是要推送到这个service的，Envoy利用这一点，允许 Istio 将大量关于目标服务的流量行为信号作为属性提取出来，再根据Pilot分发给他的策略进行流量再分发，而这些属性又可以在 Mixer 中用于执行监控等等，以提供整个网格行为的信息。

## Istio架构

![image](https://preliminary.istio.io/docs/concepts/what-is-istio/arch.svg)
> 图片摘自官网文档

**Envoy是负责真正干活的工人，Mixer是收集信息的，Pilot是翻译兼管理，Citadel是监管人员。**

#### 各个控件的解释
Istio 使用 Envoy 代理的扩展版本，Envoy 是以 C++ 开发的高性能代理，用于调解服务网格中所有服务的所有入站和出站流量。

Mixer 是一个独立于平台的组件，负责在服务网格上执行访问控制和使用策略，并从 Envoy 代理和其他服务收集遥测数据。

Pilot 为 Envoy sidecar 提供服务发现功能，为智能路由（例如 A/B 测试、金丝雀部署等）和弹性（超时、重试、熔断器等）提供流量管理功能。

Citadel 通过内置身份和凭证管理赋能强大的服务间和最终用户身份验证。

详情请参考[Istio中文文档](https://preliminary.istio.io/zh/docs/concepts/what-is-istio/#citadel)

## Istio主要用途
- 金丝雀版本的尝试，和将含有某部分共同特征的特定用户信息分发到特定pod。
- 金丝雀这是真的非常实用，新版本迭代的时候心里总是会缺少一点底气，会紧张，部署金丝雀，观察之后再部署，部署体验和发量直线上升，就算影响一小部分用户也不会至于背锅。
- 特定用户分发可以给一下类似vip之类的用户更好的响应体验。
- 最舒服的是不用改一行业务代码就可以接入这些功能。
- 对外服务增加重试，熔断，设置超时等机制。
- 利用Gateway配置对流量进行重定向，将流量流至https，或者流向其他指定的服务。


## Istio流量配置

#### 配置内容

1. VirtualService 在 Istio 服务网格中定义路由规则，控制路由如何路由到服务上。
2. DestinationRule 是 VirtualService 路由生效后，配置应用与请求的策略集
3. ServiceEntry 是通常用于在 Istio 服务网格之外启用对服务的请求。
4. Gateway 为 HTTP/TCP 流量配置负载均衡器，最常见的是在网格的边缘的操作，以启用应用程序的入口流量。

四种配置看起来有点复杂，其实很并不是特别复杂。

VirtualService：路由规则，定义一些规则，告诉符合这些规则的流量改流该哪个DestinationRule定义的策略，由subset属性指定。

DestinationRule：策略集，分配到这里的流量我想搞点什么骚操作都可以，声明labels：v1，那就会流到具有标签为v1的pop中，设定负载模式，限制连接数等等，再在里面进行流量分发都可以。

ServiceEntry：将非托管的服务也纳入Istio的管理，比如他声明一个host之后，比如声明*.foo.com可访问，既然是host，也可以跟VirtualService 以及 DestinationRule 配合工作。

Gateway：暴露一个端口，对外部进来的流量进行转发又或者重定义，改写。在VirtualService中声明绑定gateways之后，又和VirtualService 以及 DestinationRule 配合工作了。

详情请参考[具体配置信息](https://preliminary.istio.io/zh/docs/concepts/what-is-istio/#citadel)

## Istio安全配置

Istio对安全控制也是非常棒并且可配置的。

四个组件：
1. Citadel是储存证书和密钥，需要最安全的保护的地方。
2. Pilot 管理安全策略并下发给Sidecar
3. Sidecar和周边代理拿到安全策略之后相互从Citadel拿证书和密钥去通讯。
4. Mixer能提供登录插件监控插件去进行扩展。

#### 如何确定服务器身份
Istio需要去确定每个服务器的身份。他是利用SPIFFE来向服务发布身份的。通讯中分为传输身份和来源身份，传输身份是指在两个服务之前相互调用时彼此的身份，来源身份是客户端的身份，如账户，cookic，让我们能找到指定的用户。

#### 了解一些基本概念之后我们来看看istio是如何保证通讯是如何安全进行的。

既然存在Citadel这样的证书下发机构，那服务之间的通讯肯定支持TLS认证来保证通讯的安全。当然这是对我们的代码也是无侵入性的。在容器里，Envoy将出站流量catch之后，将流量重新路由到sideCar Envoy，然后Server端也有一个Envoy，通讯双方的Envoy建立一个双向的TLS链接，从而进行安全通讯。

那命名身份又有什么作用呢，Istio认识了每一台服务器的身份之后，通过K8s的apiserver，pilot会知道哪个服务是部署在哪台服务器的，这样就算恶意用户获取了服务器代码起了一台假的服务器，并且入侵了dns将我们的客户端服务器请求到假服务器。Envoy也会检查服务器的标识是否允许允许该服务，如果不允许，则Envoy会拒绝该请求。

来源身份认证则是用jwt来确认每个来源身份的身份信息。从而针对来源身份去限制用户的操作。

#### 具体的配置内容

通讯方式的认证策略，配置范围有整个网格范围的配置，

```
apiVersion: "authentication.istio.io/v1alpha1"
kind: "MeshPolicy"
metadata:
  name: "default"
spec:
  peers:
  - mtls: {}
```

针对某个命名空间的配置

```
apiVersion: "authentication.istio.io/v1alpha1"
kind: "Policy"
metadata:
  name: "default"
  namespace: "ns1"
spec:
  peers:
  - mtls: {}
```
针对某个服务的配置

```
apiVersion: "authentication.istio.io/v1alpha1"
kind: "Policy"
metadata:
  name: "reviews"
spec:
  targets:
  - name: reviews
    peers:
  - mtls: {}
```
对于每项服务，Istio 都应用最窄的匹配策略。顺序是：特定服务>命名空间范围>网格范围。尽量避免同个范围多个配置，不然就会随机取一个。

里面的peers就是开启TLS认证的方法。

#### 来源认证配置：

要让一个服务启用来源认证配置需要先为该命名空间启用Istio授权之后再配置角色，指定服务所属角色，挺麻烦的，其实使用我们各自的权限认证可能更加方便轻松。

## 策略与遥测

在这里，终于要介绍Mixer模块的功能了。

Mixer可以做到后端抽象和中介的功能，这两个名词都比较抽象，其实简单来讲就是将一些插件和Mixer配合从而做到一些运维功能，比如日志，监控，配额等等。

既然是要进行日志什么的，那么Istio必须提供数据让我们去处理或者上报，而这些上报的信息称做属性，类似于这样的值：
```
request.path: xyz/abc
request.size: 234
request.time: 12:34:56.789 04/17/2017
source.ip: 192.168.0.1
destination.service: example
```
每个Envoy都会上报这些属性集给Mix进行处理，有用到针对数据库少读多写的场景的一个策略，要知道，每个pop都有一个Envoy存在，如果一个Envoy存了很多数据的话，占用pop的内存也会变多，所以Envoy先作为一个一级缓存，并不存很多数据，然后Envoy将数据集合起来再去发送给Mixer，然后再由Mixer与各个运维插件进行通信。从而减少通讯次数，优化传输效率。

#### 具体配置内容
需要三个配置，分别是：
1. 处理器：其实就是想接入的运维插件的一些配置，如Prometheus。
2. 实例：处理属性，以调用处理器。
3. 规则：其实就是分发器，用来决定哪些实例使用哪些处理器。
