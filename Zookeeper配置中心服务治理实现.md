## Zookeeper配置中心服务治理实现
### 背景
上一篇博客详细讲解了为什么我们要选择Zookeeper作为服务发现框架而不是使用Eureka。
[参考](https://blog.csdn.net/qq_24210767/article/details/81713075) 这篇博客将继续讲解怎么实现的，里面会有大量的代码的拷贝，具体的架构可以参考上一篇博客的架构演进图。本篇主要讲解实现过程。

### 准备工作
1. Spring Maven项目添加Zookeeper的依赖包,这样就可以使用Zookeeper的SDK了。

```
 		<dependency>
            <groupId>org.apache.zookeeper</groupId>
            <artifactId>zookeeper</artifactId>
            <version>3.3.6</version>
       </dependency>
        
```


### football-server实现服务注册功能
 创建Zookeeper.properties在resources目录下，并写入参数。

```
server.parent.name = football (注册服务的父路径)
server.instance.name = football-server (注册服务的名字)
server.port = 8081 (注册服务的端口号)
zookeeper.host = Your zookeeper address
zookeeper.port = Your zookeeper port 
```

 创建Zookeeper.properties读取文件的类。

```
@Component
public class ZookeeperProperties {

    @Value("${zookeeper.host}")
    private String host;

    @Value("${zookeeper.port}")
    private String zkPort;

    @Value("${server.parent.name}")
    private String parentName;

    @Value("${server.port}")
    private String serverPort;

    @Value("${server.instance.name}")
    private String instanceName;

    public String getZookeeperAddress(){

        return String.format("%s:%s",host,zkPort);
    }

    public String getParentName() {
        parentName=parentName.replace("/","");
        return parentName;
    }

    public String getServerPort() {
        return serverPort;
    }

    public String getInstanceName() {
        parentName=parentName.replace("/","");
        return instanceName;
    }
}

```

在服务启动成功后，调用Zookeeper的JavaSDK向Zookeeper注册自己。

```

@Service
public class ZookeeperRegistrar implements ApplicationListener<ContextRefreshedEvent> {


    @Autowired
    ZookeeperProperties zookeeperProperties;



    private static Logger log = LoggerFactory.getLogger(ZookeeperRegistrar.class);


    @Override
    public void onApplicationEvent(ContextRefreshedEvent contextRefreshedEvent) {
        if(contextRefreshedEvent.getApplicationContext().getParent()==null){
            try {
                registry();
            } catch (IOException e) {
                log.error("fooball-server registry to zookeeper fault ,{}",e.getMessage());
                e.printStackTrace();
            } catch (InterruptedException e) {
                log.error("fooball-server registry to zookeeper fault ,{}",e.getMessage());
                e.printStackTrace();
            } catch (KeeperException e) {
                log.error("fooball-server registry to zookeeper fault ,{}",e.getMessage());
                e.printStackTrace();
            }

        }
    }

    public void registry() throws IOException, KeeperException, InterruptedException {

        Watcher watcher=new Watcher() {
            @Override
            public void process(WatchedEvent watchedEvent) {
                log.info("zookeeper watcher ,{}",watchedEvent.toString());
            }
        };
        ZooKeeper zooKeeper = new ZooKeeper(zookeeperProperties.getZookeeperAddress(),200000,watcher);
        initParentPath(zooKeeper,watcher);
        createInstancePath(zooKeeper,watcher);

    }

    public void initParentPath(ZooKeeper zookeeper,Watcher watcher) throws KeeperException, InterruptedException, UnknownHostException {
        String parentPath = String.format("/%s",zookeeperProperties.getParentName());
        log.info("fooball-server parentPath:{}",parentPath);
        if(zookeeper.exists(parentPath,watcher)==null){
            zookeeper.create(parentPath,null,ZooDefs.Ids.OPEN_ACL_UNSAFE,CreateMode.PERSISTENT);
            log.info("create parentPath {} success.",parentPath);
        }

    }

    public void createInstancePath(ZooKeeper zooKeeper,Watcher watcher) throws KeeperException, InterruptedException, UnknownHostException {
        String instancePath = String.format("/%s/%s",zookeeperProperties.getParentName(),zookeeperProperties.getInstanceName());
        zooKeeper.create(instancePath,instanceInfoToByte(),ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
        log.info("create path {} success.",zookeeperProperties.getInstanceName());
    }


    public String getHost() throws UnknownHostException {
        InetAddress address = InetAddress.getLocalHost();
        return address.getHostAddress();
    }

    public InstanceInfo createInstanceInfo() throws UnknownHostException {
        String host = String.format("%s:%s",getHost(),zookeeperProperties.getServerPort());
        log.info("current instance host {}",host);
        return new InstanceInfo(zookeeperProperties.getInstanceName(),host);
    }

    public byte[] instanceInfoToByte() throws UnknownHostException {
        Gson gson=new Gson();
        InstanceInfo instanceInfo = createInstanceInfo();
        return gson.toJson(instanceInfo).getBytes();
    }



}



```

### fooball-meta实现服务发现功能
创建Zookeeper.properties和创建读取.properties文件的类和football-server一致。

从Zookeeper获取注册服务的参数

```
@Service
public class MetaService implements IMetaService {

    @Autowired
    ZooKeeper zooKeeper;

    @Autowired
    ZookeeperProperties zookeeperProperties;

    @Autowired
    Watcher watcher;

    private static Logger log = LoggerFactory.getLogger(MetaService.class);

    @Override
    public List<InstanceInfo> getServiceInfo() {
        try {
            String parentPath = String.format("/%s",zookeeperProperties.getParentName());
            List<String> childs = zooKeeper.getChildren(parentPath,false);
            return getInstancesByName(childs);
        } catch (KeeperException e) {
            log.error("Zookeeper getChildren error!");
            e.printStackTrace();
        } catch (InterruptedException e) {
            log.error("Zookeeper getChildren error!");
            e.printStackTrace();
        }
        return Collections.emptyList();
    }

    public List<InstanceInfo> getInstancesByName(List<String> childs) throws KeeperException, InterruptedException {
        List<InstanceInfo> instanceInfos =new ArrayList<InstanceInfo>();
        for(String childName:childs){
            String childPath = String.format("/%s/%s",zookeeperProperties.getParentName(),childName);
            instanceInfos.add(getInstanceFromZK(childPath));
        }
        return instanceInfos;
    }

    public InstanceInfo getInstanceFromZK(String path) throws KeeperException, InterruptedException {

        byte[] data = zooKeeper.getData(path,watcher,null);
        String instanceJson = new String(data);
        return new Gson().fromJson(instanceJson,InstanceInfo.class);
    }

}

```





