Consul 简化了分布式环境中的服务的注册和发现流程，通过 HTTP 或者 DNS 接口发现。支持外部 SaaS 提供者等。

consul提供的一些关键特性：
1,service discovery：consul通过DNS或者HTTP接口使服务注册和服务发现变的很容易，一些外部服务，例如saas提供的也可以一样注册。
2,health checking：健康检测使consul可以快速的告警在集群中的操作。和服务发现的集成，可以防止服务转发到故障的服务上面。
3,key/value storage：一个用来存储动态配置的系统。提供简单的HTTP接口，可以在任何地方操作。
4,multi-datacenter：无需复杂的配置，即可支持任意数量的区域(数据中心)。

要想利用consul提供的服务实现服务的注册与发现，我们需要建立consul cluster。
在consul方案中，每个提供服务的节点上都要部署和运行consul的agent，所有运行consul agent节点的集合构成consul cluster。
consul agent有两种运行模式：server和client。这里的server和client只是consul集群层面的区分，与搭建在cluster之上的应用服务无关。
以server模式运行的consul agent节点用于维护consul集群的状态，
官方建议每个consul cluster至少有3个或以上的运行在server mode的agent，client节点不限。
我们这里以安装三个节点为例，环境配置如下
192.168.1.100 以server模式运行
192.168.1.101，192.168.1.102 以client模式运行

Consul服务注册:
可以通过service definition服务定义或者Http Api来注册服务。
service definition是一种常用的注册服务的方式，
1，为Consul配置创建一个directory，consul会加载这个目录下所有的配置文件。
  sudo mkdir /etc/consul.d  （.d 表示这个这是个文件夹）
2，写入服务定义
  创建一个服务名为web，运行在80端口。另外，在添加一个可用用来查询服务的标签。
  {
    "servie" : {
        "name": "web",
        "tags": ["rails"],
        "port": 80
    }
  }
  echo '{"service": {"name": "web", "tags": ["rails"], "port": 80}}' \
    | sudo tee /etc/consul.d/web.json
3，用consul definition开启服务
  ./consul agent -dev -config-dir=/etc/consul.d

4,通过http api来查询服务
curl http://localhost:8500/v1/catalog/service/web
curl http://localhost:8500/v1/health/service/web?passing

Consul cluster
通过vagrant构建虚拟机实现
Vagrantfile总定义了两个node，并配置好了consul
1, 开启定义好的节点
vagrant up
2，开始配置节点
  通过ssh进入node，开始集群的配置
  vagrant ssh n1 进入节点n1
  consul agent -server -bootstrap-expect=1 \
    -data-dir=/tmp/consul -node=agent-one -bind=172.20.20.10 \
    -config-dir=/etc/consul.d
  集群中的每一个node都必须有一个unique name,consul默认使用node的hostname作为这个节点的名称。我们可用通过-node选项重写。我们还需要指定一个bind address，这是consul要监听的，并且是对于集群中所有节点开放的可用的.尽管bind address不是必须的，最后还是提供一个。Consul会默认金婷所有的ipv4，但是如果有multiiple private ips就会启动失败。
  -server 表明是server

  vagrant ssh n2
  consul agent -data-dir=/tmp/consul -node=agent-two \
    -bind=172.20.20.11 -config-dir=/etc/consul.d
然后进入server node，手动置顶关系
vagrant ssh n1
consul join 172.20.20.11(需要加入的node的ip)
然后可以通过consul members来查看consul cluster
可以通过-join来配置server client
eg：consul agent -bind=172.20.20.11 -client=172.20.20.11 -data-dir=data -node=test2 -join=172.20.20.10

consul leave -rpc-addr=192.168.1.100:8400

Consul Health Checks
1,Defining Checks，
  定义一个要注册的check definition，也可以使用http api
  echo '{"check": {"name": "ping",
  "script": "ping -c1 google.com >/dev/null", "interval": "30s"}}' \
  >/etc/consul.d/ping.json
  echo '{"service": {"name": "web", "tags": ["rails"], "port": 80,
  "check": {"script": "curl localhost >/dev/null 2>&1", "interval": "10s"}}}' \
  >/etc/consul.d/web.json
2，查看健康状态
curl http://localhost:8500/v1/health/state/critical

Consul KV data
出服务发现以及集成健康检查外，consul还提供了kv store。kv data可以用于保存动态配置，辅助服务协同service coordination，构建领导选举leader election等.
可以通过Http api或者consul kv cli来操作k/v store.
consul kv put redis/config/minconns 1
consul kv put redis/config/maxconns 25
consul kv put -flags=42 redis/config/users/admin abcd1234
consul kv get redis/config/minconns
consul kv get -detailed redis/config/minconns -detailed 显示metadata about the field

 consul kv get -recurse 列出所有的key
 consul kv delete redis/config/minconns 删除key
consul kv delete -recurse redis 删除所有前缀为redis的key

consul kv put -cas -modify-index=123 foo bar 如果-modify-index=123在设置
modify-index可以通过-detailed查询
