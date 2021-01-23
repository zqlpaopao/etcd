#  1、集群协调服务的标准

> 1. 可用性角度,高可用
>
>    协调服务作为集群的控制面存储，它保存了各个服务的部署、运行信息。若它故障，可能会导致集群无法变更、服务副本数无法协调。业务服务若此时出现故障，无法创建新的副本，可能会影响用户数据面。
>
> 2. 数据一致性角度:提供读取“最新”数据的机制
>
> 3. 容量角度:低容量、仅存储、仅存储关键元数据配置
>
> 4. 功能:增删改查,监听数据变化的机制
>
> 5. 运维复杂性,可维护性



# 2、etc d 存储模式

简单内存树

![image-20210120205915039](geektime-etcd.assets/image-20210120205915039.png)



# 3、V2 和V3的对比

## A、V2版本

1. <font color=red size=5x>V2 不支持范围查询(模糊匹配)和分页查询</font>
2. <font color=green size=5x>V2 不支持Watch的可靠性,V2是内存型,不支持key的历史版本问题</font>
3. <font color=red size=5x>V2是基于http1.x的长连接,不支持多路复用,没有http的压缩功能</font>
4. <font color=green size=5x>V2 不支持统一过期时间的key的设置,要为每个key设置过期时间</font>
5. <font color=red size=5x>V2 在内存中维护一棵树存储所有key和value,同步磁盘回消耗大量CPU资源和磁盘I/O资源</font>
6. <font color=green size=5x>V2 不支持范围查询(模糊匹配)和分页查询</font>



## B、V3版本

1. <font color=green size=5x>V3 通过引入B- tree、boltDb实现MVCC数据库,降低内存使用率</font>
2. <font color=red size=5x>V3 实现事务,支持多个key的原子更新</font>
3. <font color=green size=5x>V3 使用了gRPC API,降低json格式的开销</font>
4. <font color=red size=5x>V3 支持http2的多路复用,减少watch下的连接数</font>
5. <font color=green size=5x>V3 lease优化TTl的问题,每个lease具有一个TTL,相同的TTL的key随着lease的过期而过期</font>
6. <font color=red size=5x>V3支持范围查询(模糊匹配)和分页查询</font>



# 4、V3基础架构

## A、架构图

![image-20210123111820468](geektime-etcd.assets/image-20210123111820468.png)



> 1、client层 
>
> client 层包括 client v2 和 v3 两个大版本 API 客户端库，提供了简洁易用的 API，同时支持负载均衡、节点间故障自动转移，可极大降低业务使用 etcd 复杂度，提升开发效率、服务可用性。

> 2、API网络层
>
> client和server的通信协议
>
> V2和V3的api，==V3通过 grpc的 grpc-getWay组件支持HTTP1.x协议==
>
> ==server间通过Raft算法实现的数据复制和Leader的选举使用的HTTP2协议==

> 3、raft算法层
>
> Raft算法层实现了Leader选举、日志复制、RadeIndex等核心算法、保证etcd多个节点间的数据一致性、提升服务可用性等，是etcd的基石和亮点

> 4、功能逻辑层
>
> etcd核心特性实现层，KVServer模块、MVCC模块、AUth鉴权模块、Lease租约模块、Compactor压缩模块，==其中MVCC是由treeIndex模块和boltdb模块组成==

> 5、存储层
>
> 存储层 ：存储层含预写日志（WAL）模块、快照（Snapshot）模块、botdb模块，其中WAL是保证etcd crash后数据不丢失，boltdb保证数据的持久化



## B、KVServer

client 发送 Range RPC 请求到了 server 后，就开始进入我们架构图中的流程二，也就是 KVServer 模块了。

etcd 提供了丰富的 metrics、日志、请求行为检查等机制，可记录所有请求的执行耗时及错误码、来源 IP 等，也可控制请求是否允许通过，比如 etcd Learner 节点只允许指定接口和参数的访问

### 拦截器

<font color=red size=5x>要求执行一个操作前集群必须有 Leader，防止脑裂；</font>

<font color=r size=5x>**请求延时超过指定阈值的，打印包含来源 IP 的慢查询日志 (3.5 版本)**</font>

实现方法的非入侵式的、日志、请求行为等检查机制

etcd server 定义了如下的 Service KV 和 Range 方法，启动的时候它会将实现 KV 各方法的对象注册到 gRPC Server，并在其上注册对应的拦截器。下面的代码中的 Range 接口就是负责读取 etcd key-value 的的 RPC 接口。

```go
service KV {  
  // Range gets the keys in the range from the key-value store.  
  rpc Range(RangeRequest) returns (RangeResponse) {  
      option (google.api.http) = {  
        post: "/v3/kv/range"  
        body: "*"  
      };  
  }  
  ....
}  
```





## C 、线性读流程-默认

![image-20210123112955399](geektime-etcd.assets/image-20210123112955399.png)



### 数据不一致场景

1、 <font color=red size=5x>**client 发起请求**</font>

2、<font color=red size=5x>**client 发起请求**</font>

3、<font color=red size=5x>**client 发起请求**</font>

4、<font color=red size=5x>**client 发起请求**</font>

5、<font color=red size=5x>**client 发起请求**</font>

6、<font color=red size=5x>**client 发起请求**</font>











# 5、环境准备

```go
go get github.com/mattn/goreman
```

下载 etcd v3.4.9 二进制文件

```go
https://github.com/etcd-io/etcd/releases/v3.4.9
```

启动集群

goreman -f Procfile start命令就可以快速启动一个 3 节点的本地集群了。

```go
goreman -f Procfile start
```



clientv3 库基于 gRPC client API 封装了操作 etcd KVServer、Cluster、Auth、Lease、Watch 等模块的 API，同时还包含了负载均衡、健康探测和故障切换等特性。



如果你的 client 版本 <= 3.3，那么当你配置多个 endpoint 时，负载均衡算法仅会从中选择一个 IP 并创建一个连接（Pinned endpoint），这样可以节省服务器总连接数。但在这我要给你一个小提醒，在 heavy usage 场景，这可能会造成 server 负载不均衡。在 client 3.4 之前的版本中，负载均衡算法有一个严重的 Bug：如果第一个节点异常了，可能会导致你的 client 访问 etcd server 异常，特别是在 Kubernetes 场景中会导致 APIServer 不可用。不过，该 Bug 已在 Kubernetes 1.16 版本后被修复。













