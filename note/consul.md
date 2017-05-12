consul的生命周期

到我们在Spring Cloud应用中使用Consul来实现服务治理时，Consul不会自动将不可用服务实例注销，使得在实际使用过程中可能因为一些操作失误，环境变更等原因让Consul中存在一些无效实例信息，而这些实例在Consul中会长期存在，并处于断开状态。它们虽然不会影响到正常的服务消费过程，但是会干扰我们的监控，所以我们可以实现一个清理接口，在确认故障实例可以清理的时候进行调用来讲这些信息清理掉。

Spring Cloud Consul在关闭程序时候实现的注销方法，具体如下：
public class ConsulLifecycle extends AbstractDiscoveryLifecycle {
    ...
    private void deregister(String serviceId) {
        if (!this.properties.isRegister()) {
            return;
        }
        if (ttlScheduler != null) {
            ttlScheduler.remove(serviceId);
        }
        log.info("Deregistering service with consul: {}", serviceId);
        client.agentServiceDeregister(serviceId);
    }
    ...
}
我们可以看到，当应用关闭时候的注销操作是通过调用client.agentServiceDeregister(serviceId)来实现的。其中client是consul-api的com.ecwid.consul.v1.ConsulClient实例。而agentServiceDeregister方法则是对/v1/agent/service/deregister/<serviceID> 接口的实现，该接口主要用来从Consul Agent中根据serviceId来注销实例。

以此实现为范例，于是开始的思路是这样的：

先通过consulClient.getHealthServices(serviceId, false, null)根据serviceId来获取服务实例清单
遍历实例清单中有不是PASSING状态的实例，就调用client.agentServiceDeregister(serviceId)来剔除
具体实现如下：
@RestController
public class ApiController {
    @Autowired
    private ConsulClient consulClient;
    @RequestMapping(value = "/unregister/{id}", method = RequestMethod.POST)
    public String unregisterServiceAll(@PathVariable String id) {
        List<HealthService> response = consulClient.getHealthServices(id, false, null).getValue();
        for(HealthService service : response) {
            service.getChecks().forEach(check -> {
                if(!check.getStatus().name().equals(Check.CheckStatus.PASSING.name())) {
                    logger.info("unregister : {}", check.getServiceId());
                    consulClient.agentServiceDeregister(check.getServiceId());
                }
            });
        }
        return null;
    }
}
但是，在测试后发现该方法只能剔除同一个agent上的非PASSING实例。

Catalog误区
继续搜索了一下Consul的文档，发现了这个接口：/v1/catalog/deregister : Deregisters a node, service, or check。于是，尝试了用该接口来替换之前的consulClient.agentServiceDeregister(check.getServiceId());实现。
CatalogDeregistration catalogDeregistration = new CatalogDeregistration();
catalogDeregistration.setDatacenter("dc1");
catalogDeregistration.setNode(check.getNode());
catalogDeregistration.setServiceId(check.getServiceId());
catalogDeregistration.setCheckId(check.getCheckId());
consulClient.catalogDeregister(catalogDeregistration);
经过测试，该方法可以实现短暂的剔除，但是过一段时间之后这些被剔除的实例又都恢复回来了

服务实例只能在注册的Agent上进行注销，那么我们的实现完全可以按照该思路来实现，方法很简单，只需要对一开始实现的内容做一些调整，依然使用client.agentServiceDeregister(serviceId)方法，只是我们需要调整client连接的agent必须是serviceId注册的agent。所以，最终的修改结果如下：
List<HealthService> response = consulClient.getHealthServices(id, false, null).getValue();
for(HealthService service : response) {
    // 创建一个用来剔除无效实例的ConsulClient，连接到无效实例注册的agent
    ConsulClient clearClient = new ConsulClient(service.getNode().getAddress(), 8500);
    service.getChecks().forEach(check -> {
        if(check.getStatus() != Check.CheckStatus.PASSING) {
            logger.info("unregister : {}", check.getServiceId());
            clearClient.agentServiceDeregister(check.getServiceId());
        }
    });
}

基于Consul的分布式锁实现
构建分布式系统的时候，进场需要控制对共享资源的互斥访问。这个时候就要涉及到分布式锁的实现。我们可以给予Consul的key/value存储来实现分布式锁以及信号量的方法。
分布式锁实现：
基于Consul的分布式锁主要利用key/value存储api中的acquire和release操作来实现，acquire和release操作室类似与check-and-set的操作:
1,acquire操作只有当锁不存在持有者时才会返回true，并且set设置的value值，同时执行操作的session会持有该key的锁，否者就返回false
2,release操作则是使用指定的session来释放某个key的锁，如果指定的session无效，那么会返回false，否则就会set设置value值，并返回true
实现中需要使用的key/value的api
create session
delete session
KV acquire/release

public class Lock {

    private static final String prefix = "lock/";  // 同步锁参数前缀

    private ConsulClient consulClient;
    private String sessionName;
    private String sessionId = null;
    private String lockKey;

    /**
     *
     * @param consulClient
     * @param sessionName   同步锁的session名称
     * @param lockKey       同步锁在consul的KV存储中的Key路径，会自动增加prefix前缀，方便归类查询
     */
    public Lock(ConsulClient consulClient, String sessionName, String lockKey) {
        this.consulClient = consulClient;
        this.sessionName = sessionName;
        this.lockKey = prefix + lockKey;
    }

    /**
     * 获取同步锁
     *
     * @param block     是否阻塞，直到获取到锁为止
     * @return
     */
    public Boolean lock(boolean block) {
        if (sessionId != null) {
            throw new RuntimeException(sessionId + " - Already locked!");
        }
        sessionId = createSession(sessionName);
        while(true) {
            PutParams putParams = new PutParams();
            putParams.setAcquireSession(sessionId);
            if(consulClient.setKVValue(lockKey, "lock:" + LocalDateTime.now(), putParams).getValue()) {
                return true;
            } else if(block) {
                continue;
            } else {
                return false;
            }
        }
    }

    /**
     * 释放同步锁
     *
     * @return
     */
    public Boolean unlock() {
        PutParams putParams = new PutParams();
        putParams.setReleaseSession(sessionId);
        boolean result = consulClient.setKVValue(lockKey, "unlock:" + LocalDateTime.now(), putParams).getValue();
        consulClient.sessionDestroy(sessionId, null);
        return result;
    }

    /**
     * 创建session
     * @param sessionName
     * @return
     */
    private String createSession(String sessionName) {
        NewSession newSession = new NewSession();
        newSession.setName(sessionName);
        return consulClient.sessionCreate(newSession, null).getValue();
    }

}

通过使用分布式锁的形式来控制并发，多个同步操作只会有一个操作能够被完成，其他操作只有在等锁释放之后才有机会执行，所以通过这样的分布式锁，我们可以控制共享资源同时只能被一个操作锁执行，以保障数据处理时的分布式并发问题。

需要实现对锁的超时清理等控制，保证及时出现了未正常解锁的情况下也能自动修复，以提升系统的健壮性。

基于Consul的分布式信号量实现
信号量是我们在实现并发控制时经常使用的手段，主要用来限制同时并发线程或者进程的数据量，比如：zuul默认情况下就使用信号量来限制每个路由的并发数，以实现不同路由间的资源隔离。
实现思路：
信号量存储：semaphore/key
acquire操作：
创建 session
锁定key竞争者:semaphore/key/session
查询信号量:semaphore/key/.lock,可以获得如下内容（如果不是第一次创建信号量，将获取不到，这个时候就直接创建）
{
    "limit": 3,
    "holders": [
        "90c0772a-4bd3-3a3c-8215-3b8937e36027",
        "93e5611d-5365-a374-8190-f80c4a7280ab"
    ]
}
1，如果持有者已达上线，则返回false，如果阻塞模式，就继续尝试acquire操作
2，如果持有者未达上线，更新semaphore/key/.lock内容，将当前线程的sessionId加入到holders中；更新的时候需要设置cas，它的值是查询信号量步骤获得的ModifyIndex值，该值用于保证更新操作的基础没有被其他竞争者更新。如果更新成功，就开始执行具体逻辑。如果没有更新成功，说明有其他竞争者抢占了资源，返回false，阻塞模式下继续尝试acquire操作
3，release操作：  从semaphore/key/.lock的holders中移除当前sessionId，删除semaphore/key/session，删除当前的session
public class Semaphore {

    private Logger logger = Logger.getLogger(getClass());

    private static final String prefix = "semaphore/";  // 信号量参数前缀

    private ConsulClient consulClient;
    private int limit;
    private String keyPath;
    private String sessionId = null;
    private boolean acquired = false;

    /**
     *
     * @param consulClient consul客户端实例
     * @param limit 信号量上限值
     * @param keyPath 信号量在consul中存储的参数路径
     */
    public Semaphore(ConsulClient consulClient, int limit, String keyPath) {
        this.consulClient = consulClient;
        this.limit = limit;
        this.keyPath = prefix + keyPath;
    }

    /**
     * acquired信号量
     *
     * @param block 是否阻塞。如果为true，那么一直尝试，直到获取到该资源为止。
     * @return
     * @throws IOException
     */
    public Boolean acquired(boolean block) throws IOException {

        if(acquired) {
            logger.error(sessionId + " - Already acquired");
            throw new RuntimeException(sessionId + " - Already acquired");
        }

        // create session
        clearSession();
        this.sessionId = createSessionId("semaphore");
        logger.debug("Create session : " + sessionId);

        // add contender entry
        String contenderKey = keyPath + "/" + sessionId;
        logger.debug("contenderKey : " + contenderKey);
        PutParams putParams = new PutParams();
        putParams.setAcquireSession(sessionId);
        Boolean b = consulClient.setKVValue(contenderKey, "", putParams).getValue();
        if(!b) {
            logger.error("Failed to add contender entry : " + contenderKey + ", " + sessionId);
            throw new RuntimeException("Failed to add contender entry : " + contenderKey + ", " + sessionId);
        }

        while(true) {
            // try to take the semaphore
            String lockKey = keyPath + "/.lock";
            String lockKeyValue;

            GetValue lockKeyContent = consulClient.getKVValue(lockKey).getValue();

            if (lockKeyContent != null) {
                // lock值转换
                lockKeyValue = lockKeyContent.getValue();
                BASE64Decoder decoder = new BASE64Decoder();
                byte[] v = decoder.decodeBuffer(lockKeyValue);
                String lockKeyValueDecode = new String(v);
                logger.debug("lockKey=" + lockKey + ", lockKeyValueDecode=" + lockKeyValueDecode);

                Gson gson = new Gson();
                ContenderValue contenderValue = gson.fromJson(lockKeyValueDecode, ContenderValue.class);
                // 当前信号量已满
                if(contenderValue.getLimit() == contenderValue.getHolders().size()) {
                    logger.debug("Semaphore limited " + contenderValue.getLimit() + ", waiting...");
                    if(block) {
                        // 如果是阻塞模式，再尝试
                        try {
                            Thread.sleep(100L);
                        } catch (InterruptedException e) {
                        }
                        continue;
                    }
                    // 非阻塞模式，直接返回没有获取到信号量
                    return false;
                }
                // 信号量增加
                contenderValue.getHolders().add(sessionId);
                putParams = new PutParams();
                putParams.setCas(lockKeyContent.getModifyIndex());
                boolean c = consulClient.setKVValue(lockKey, contenderValue.toString(), putParams).getValue();
                if(c) {
                    acquired = true;
                    return true;
                }
                else
                    continue;
            } else {
                // 当前信号量还没有，所以创建一个，并马上抢占一个资源
                ContenderValue contenderValue = new ContenderValue();
                contenderValue.setLimit(limit);
                contenderValue.getHolders().add(sessionId);

                putParams = new PutParams();
                putParams.setCas(0L);
                boolean c = consulClient.setKVValue(lockKey, contenderValue.toString(), putParams).getValue();
                if (c) {
                    acquired = true;
                    return true;
                }
                continue;
            }
        }
    }

    /**
     * 创建sessionId
     * @param sessionName
     * @return
     */
    public String createSessionId(String sessionName) {
        NewSession newSession = new NewSession();
        newSession.setName(sessionName);
        return consulClient.sessionCreate(newSession, null).getValue();
    }

    /**
     * 释放session、并从lock中移除当前的sessionId
     * @throws IOException
     */
    public void release() throws IOException {
        if(this.acquired) {
            // remove session from lock
            while(true) {
                String contenderKey = keyPath + "/" + sessionId;
                String lockKey = keyPath + "/.lock";
                String lockKeyValue;

                GetValue lockKeyContent = consulClient.getKVValue(lockKey).getValue();
                if (lockKeyContent != null) {
                    // lock值转换
                    lockKeyValue = lockKeyContent.getValue();
                    BASE64Decoder decoder = new BASE64Decoder();
                    byte[] v = decoder.decodeBuffer(lockKeyValue);
                    String lockKeyValueDecode = new String(v);
                    Gson gson = new Gson();
                    ContenderValue contenderValue = gson.fromJson(lockKeyValueDecode, ContenderValue.class);
                    contenderValue.getHolders().remove(sessionId);
                    PutParams putParams = new PutParams();
                    putParams.setCas(lockKeyContent.getModifyIndex());
                    consulClient.deleteKVValue(contenderKey);
                    boolean c = consulClient.setKVValue(lockKey, contenderValue.toString(), putParams).getValue();
                    if(c) {
                        break;
                    }
                }
            }
            // remove session key

        }
        this.acquired = false;
        clearSession();
    }

    public void clearSession() {
        if(sessionId != null) {
            consulClient.sessionDestroy(sessionId, null);
            sessionId = null;
        }
    }

    class ContenderValue implements Serializable {

        private Integer limit;
        private List<String> holders = new ArrayList<>();

        public Integer getLimit() {
            return limit;
        }

        public void setLimit(Integer limit) {
            this.limit = limit;
        }

        public List<String> getHolders() {
            return holders;
        }

        public void setHolders(List<String> holders) {
            this.holders = holders;
        }

        @Override
        public String toString() {
            return new Gson().toJson(this);
        }

    }

}
