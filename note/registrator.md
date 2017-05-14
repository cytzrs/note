获QU取最新的registrator
docker pull gliderlabs/registrator:latest
1，先运行一个Consul
docker run -d --name=consul --net=host gliderlabs/consul-server -bootstrap
curl http://127.0.0.1:8500/v1/catalog/services
{
    "consul": [],
    "gitlab-ce-22": [],
    "gitlab-ce-80": [],
    "jenkins-8080": [],
    "web": [
        "rails"
    ]
}
2，运行registrator
Registrator会在每一个主机运行
sudo docker run -d --name=registrator_1 --net=host --volume=/var/run/docker.sock:/tmp/docker.sock gliderlabs/registrator:latest consul://localhost:8500
-d run the container detached
--name name it
-net 指定网络为host模式，确保Registrator使用真是的IP
--volume=/var/run/docker.sock:/tmp/docker.sock 运行Registrator使用Docker api
其他Registrator Options:
Option 	Since 	Description
-cleanup 	v7 	Cleanup dangling services
-deregister <mode> 	v6 	Deregister exited services "always" or "on-success". Default: always
-internal 		Use exposed ports instead of published ports
-ip <ip address> 		Force IP address used for registering services
-resync <seconds> 	v6 	Frequency all services are resynchronized. Default: 0, never
-retry-attempts <number> 	v7 	Max retry attempts to establish a connection with the backend
-retry-interval <milliseconds> 	v7 	Interval (in millisecond) between retry-attempts
-tags <tags> 	v5 	Force comma-separated tags on all registered services
-ttl <seconds> 		TTL for services. Default: 0, no expiry (supported backends only)
-ttl-refresh <seconds> 		Frequency service TTLs are refreshed (supported backends only)
-useIpFromLabel <label> 		Uses the IP address stored in the given label, which is assigned to a container, for registration with Consul
consul://localhost:8500 连接consul 这个参数是Registrator自己的，是regisry URL,提供服务注册的url，我们使用localhost:8500的cosul，因为registraotr和consul运行在同样的network interface.

docker logs registrator_1
3,运行Redis
如你所见，所有提供服务的容器都会被添加到consul。
sudod docker run -d -P --name=redis Redis
-P 讲服务发布到所有的端口 to publish all ports 除了Registrator这种方式不经常使用.不只是因为发布到容器暴露的所有端口，而且还将它们随机分配到主机端口。以为Registrator和Consul是为了提供服务发现，端口无关紧要。



[WARN] consul.catalog: Register of service 'redis' on 'hostname' denied due to ACLs
这条警告辨明，consul配置了ACL token，我们需要在运行Registrator是加入-e选项-e CONSUL_HTTP_TOKEN=<your acl token>

Service Object
Regisrator首要关心的四需要添加到服务注册的服务。我们认为，a service is anything listening on a port.如果一个容器坚挺了多个端口，它就提供多个服务。

type Service struct {
    ID    string               // unique service instance ID
    Name  string               // service name
    IP    string               // IP address service is located at
    Port  int                  // port service is listening on
    Tags  []string             // extra tags to classify service
    Attrs map[string]string    // extra attribute metadata
}
docker run -d --name redis.0 -p 10000:6379 \
    -e "SERVICE_NAME=db" \
    -e "SERVICE_TAGS=master,backups" \
    -e "SERVICE_REGION=us2" progrium/redis

{
  "ID": "hostname:redis.0:6379",
  "Name": "db",
  "Port": 10000,
  "IP": "192.168.1.102",
  "Tags": ["master", "backups"],
  "Attrs": {"region": "us2"}
}


sudo docker logs registrator_1
2017/05/13 04:35:27 Starting registrator v7 ...
2017/05/13 04:35:27 Using consul adapter: consul://localhost:8500
2017/05/13 04:35:27 Connecting to backend (0/0)
2017/05/13 04:35:27 consul: current leader  127.0.0.1:8300
2017/05/13 04:35:27 Listening for Docker events ...
2017/05/13 04:35:27 Syncing services on 3 containers
2017/05/13 04:35:27 ignored: b92eb7a43c90 no published ports
2017/05/13 04:35:27 added: 418820fed6da cytzrs:gitlab:22
2017/05/13 04:35:27 added: 418820fed6da cytzrs:gitlab:80
2017/05/13 04:35:27 ignored: 418820fed6da port 443 not published on host
2017/05/13 04:35:27 added: d4ed2dc67af4 cytzrs:lonely_goodall:8080
2017/05/13 04:35:27 ignored: d4ed2dc67af4 port 50000 not published on host
2017/05/13 05:02:54 added: e1d6629f5a7e cytzrs:redis:6379


基于容器的服务注册:
Consul优势：
服务注册独立于服务本身，
主动检查服务状态
通过Registrator可以提供基于容器的服务注册
多数据中心支持
Registrator通过监控Docker事件，更具docker容器的启动和停止向注册中心发送注册和注销的消息，无需服务自身参与即可完成。
Registrator能够掌握服务真正对外开放的端口。
即使在服务无响应，甚至直接删除容器的情况下也可以完成对服务的注销。

Consul Docker服务注册与发现:
sudo docker run -d --restart=unless-stopped -p 8080:8080 rancher/server:stable
Consul集群的基本单位是agent，agent运行于主机集群中每个节点上，多个Consul agent组成集群。
Consul的两个运行模式：
Server：强一致性，保存服务状态，WAN通信
Client：无状态，转发HTTP请求，健康检查
每个数据中心建议由3-5个Server模式的agent组成高可用的集群。
大规模集群中Server模式的agent资源需求较高，可运行于专用的主机上，服务所在主机上则部署仅启用Client模式的agent。
在Rancher中新建一个应用栈Consul，添加一如下设定的服务：
数量：总是在每台主机上运行一个此容器的示例
名称：consul
选择镜像：consul
命令：agent -ui -data-dir /tmp/data -server -bootstrap-expect 3 -client 0.0.0.0 -regry-join [ServerA的地址]
-ui:开启网页GUI
-server：开启Server模式
-bootstrap-expect 3：有三个节点加入集群时开始leader选举
-client 0.0.0.0：开启client模式，绑定到所有端口
-retryjoin [ServerA的地址]：通过连接ServerA加入集群
当主机有多个IP地址时，Consul容器启动报错，提示有多个IP，请指定一个绑定IP。
对每个主机单独设置绑定IP虽然可行，但会导致每台主机的容器配置不同，不利用扩展。
Consul官方docker镜像的入口中我们发现该脚本支持使用环境变量来置顶帮顶IP。
设置容器的环境变量：CONSUL_BIND_INTERFACE=eth1
在容器启动时会自动绑定eth1端口的IP
Consul集群启动后，需要大约几十秒时间完成leader选举和状态同步，然后就可以通过8500端口打开Consul的页面查看Consul集群中节点和服务的状态。
除了以上方法，Rancher应用市场也提供了一键部署Consul集群的模板。
Consul集群搭建完成后，即可提供服务注册和发现的服务，单位了实现基于容器的服务自动注册，我们还需要安装Registrator。
和Consul agent一样，Registrator也需要在集群中每个节点部署一个容器，通过挂载docker.sock，容器中运行的Registrator可以控制主机的Docker守护进程，通过Docker events达到监控容器变化的目的，并向本节点的Consul agnet发送注册消息。
在Consul应用栈中添加服务，服务的各项设置如下：
数量：总是在每台主机上运行一个此容器的实例
名称：registrator
选择镜像：gliderlabs/registrator命令：consul://localhost:8500
网络：主机 -net=host
卷：/var/run/docker.sock:/tmp/docker.sock
此处值得注意的是要把主机的/var/run/docker.sock挂载进容器中。registrator的其他详细设定可以通过修改启动命令来修改。
完成registrator的部署后，框架的搭建就基本完成了，可以再该环境中新建容器，如果容器通过nat开放了端口，会自动被注册到consul中；在容器停止或被删除时，则会从consul中注销。
基于容器的自动注册虽然方便，但有些时候，我们需要对注册的过程进行一些定制，但为了在自动注册时加入我们自己设定的一些信息，我们需要对服务所在的容器做一些改动。
定制服务名称：
服务名称是Consul服务目录划分同一种服务的依据，在运用zuul作为服务网关进行方向代理时，如果服务列表取自consul的服务发现，服务名也是访问服务的默认路径。因此我们很有可能需要对服务名称进行定制。要设置自动注册时采用的服务名，只需要给服务容器添加标签
SERVICE_NAME=[服务名]即可，也可以通过设置环境变量SERVICE_NAME=[服务名]来达到同样的效果
配置健康检查：
我们可以通过如下方式让Registrator在注册服务的同时，注册该服务的健康检查。以Http检查为例，设置标签或环境变量SERVICE_CHECK_HTTP=/health设置健康检查路径/health;设置标签或环境变量。
SERVICE_CHECK_INTERVAL=15s设置健康检查间隔为15秒；设置标签或环境变量SERVICE_CHECK_TIMEOUT=5s设置健康检查请求超时时间为5秒。
通过上述设定，在服务被Registrator注册到Consul时，会附带注册一个间隔为15秒的http方式的健康检查，如果请求超时或失败，Consul会将服务置为故障状态，在取可用服务时不会提供该实例。
修改服务程序：
将服务运行的容器进行上述设定后（建议通过Dockerfile进行构建，直接在镜像中包含这些设置)，通过Rancher部署到之前建好的集群中运行，网络模式选择桥接，设定好开放端口，待服务启动后即可在Consul中看到服务的信息，在服务开始启动到服务能够正常的处理请求之间的这段时间，由于健康检查失败，在Consul中将显示为故障状态，直到第一次成功执行健康检查。
后续有待研究的问题有：
如何在服务发布时平滑切换，通过Consul的API设定服务为维护状态可以达到目的，但这个操作是针对agent的，有一定的局限性。
混合云环境下的跨数据中心集群如何部署。
