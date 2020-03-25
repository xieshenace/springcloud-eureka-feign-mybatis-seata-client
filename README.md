# springcloud-eureka-feign-mybatis-seata-client/server
###### 注：来源阿里云开源seata，本人只做修改
### 概览：seata由服务端及客户端组成，服务端是阿里的项目需要在后台一直运行，客户端只是集成了客户端部分配置文件而已。需要两端同时运行才可以完成分布式事务；
##### 1.整合seata的demo，此demo都配置好了，拉下来按照步骤，直接可以跑起来观察效果。

##### 2.demo项目整合Seata，主要步骤如下：(可参考进行自己项目整合)
- 1.下载服务端：原：[下载seata-server](https://github.com/seata/seata/releases)，放到linux或者自己的window/mac中；
- 2.下载客户端：此Demo项目（引入配置文件，修改配置文件(注意不要遗漏，可参考下方几个关键步骤)）
- 3.数据源代理设置
- 4.创建数据库表
- 5.启动eureka，启动server,启动client（自己的服务比如订单啊，仓库啊，等等等）

##### 关于调用成环和seata-server HA，见最后部分

### 1.此demo技术选型及版本信息

注册中心：eureka

服务间调用：feign

持久层：mybatis 

数据库：mysql 5.7.20

Springboot:2.1.7.RELEASE

Springcloud:Greenwich.SR2

jdk:1.8 

seata:1.1

使用不同组件，配置情况不同，可参考其他sample（开源地址：https://github.com/seata/seata-samples）；

### 2.demo概况
demo分为四个项目，单独启动。

- eureka:作为注册中心
- order:订单服务，用户下单后，会创建一个订单添加在order数据库，同时会扣减库存storage，扣减账户account;
- storage:库存服务，用户扣减库存；
- account:账户服务，用于扣减账户余额；

order服务关键代码如下：
```java
  /**
   * 创建订单
   *
   * @param order
   * @return 测试结果： 1.添加本地事务：仅仅扣减库存 2.不添加本地事务：创建订单，扣减库存
   */
  @Override
  @GlobalTransactional(name = "fsp-create-order", rollbackFor = Exception.class)
  public void create(Order order) {
    LOGGER.info("------->交易开始");
    // 本地方法
    orderDao.create(order);
    // 远程方法 扣减库存
    storageApi.decrease(order.getProductId(), order.getCount());
    // 模拟异常情况
    // int i = 1 / 0;
    // 远程方法 扣减账户余额
    LOGGER.info("------->扣减账户开始order中");
    accountApi.decrease(order.getUserId(), order.getMoney());
    LOGGER.info("------->扣减账户结束order中");

    LOGGER.info("------->交易结束");
  }

```
### 3.使用步骤
- 1.拉取本客户端demo代码 git clone xxxx;
- 2.[下载seata-server](https://github.com/seata/seata/releases);
- 3.执行本demo每个项目下的建表语句，resource下xx.sql文件；（undo_log.sql、account.sql、order.sql、storage.sql）;
    注：undo_log.sql需要在项目每个库都需要有一个。如果项目用的一个库则需要一个表就行；
- 4.seata相关建表语句见下文说明；

### 4.seata server（服务端）端配置信息修改
seata-server中，/conf目录下，有两个配置文件,需要结合自己的情况来修改：

##### 1.file.conf 

里面有事务组配置，锁配置，事务日志存储等相关配置信息，由于此demo使用db存储事务信息，我们这里要修改store中的配置：
```java
## **原本配置**：
## transaction log store
store {
  ## store mode: file、db
  mode = "db"   修改这里，表明事务信息用db存储

  ## file store 当mode=db时，此部分配置就不生效了，这是mode=file的配置
  file {
    dir = "sessionStore"

    # branch session size , if exceeded first try compress lockkey, still exceeded throws exceptions
    max-branch-session-size = 16384
    # globe session size , if exceeded throws exceptions
    max-global-session-size = 512
    # file buffer size , if exceeded allocate new buffer
    file-write-buffer-cache-size = 16384
    # when recover batch read size
    session.reload.read_size = 100
    # async, sync
    flush-disk-mode = async
  }

  ## database store  mode=db时，事务日志存储会存储在这个配置的数据库里
  db {
    ## the implement of javax.sql.DataSource, such as DruidDataSource(druid)/BasicDataSource(dbcp) etc.
    datasource = "dbcp"
    ## mysql/oracle/h2/oceanbase etc.
    db-type = "mysql"
    driver-class-name = "com.mysql.jdbc.Driver"
    url = "jdbc:mysql://116.62.62.26/seat-server"  修改这里
    user = "root"  修改这里
    password = "root"  修改这里
    min-conn = 1
    max-conn = 3
    global.table = "global_table"
    branch.table = "branch_table"
    lock-table = "lock_table"
    query-limit = 100
  }
}
```
**如果mode类型中选择的是db模式则：**
由于此demo我们使用db模式存储事务日志，所以，我们要创建三张表：global_table，branch_table，lock_table，建表sql在上面下载的seata-server的/conf/db_store.sql中；

由于存储undo_log是在业务库中，所以在每个业务库中，还要创建undo_log表，建表sql在/conf/db_undo_log.sql中。

由于我自定义了事务组名称，所以这里也做了修改：
```java
service {
  #vgroup->rgroup
  vgroup_mapping.fsp_tx_group = "default"  修改这里，fsp_tx_group这个事务组名称是我自定义的，一定要与client端的这个配置一致！否则会报错！
  #only support single node
  default.grouplist = "127.0.0.1:8091"   此配置作用参考:https://blog.csdn.net/weixin_39800144/article/details/100726116
  #degrade current not support
  enableDegrade = false
  #disable
  disable = false
  #unit ms,s,m,h,d represents milliseconds, seconds, minutes, hours, days, default permanent
  max.commit.retry.timeout = "-1"
  max.rollback.retry.timeout = "-1"
}
```
其他的可以先使用默认值。
如果，和本demo服务端一致使用的是file模式，则不需要做上面操作，直接使用**本Demo服务端配置**：
```java
## **本Demo服务端file.conf 配置**：
## transaction log store, only used in seata-server
store {
  ## store mode: file、db
  mode = "file"

  ## file store property
  file {
    ## store location dir
    dir = "sessionStore"
    # branch session size , if exceeded first try compress lockkey, still exceeded throws exceptions
    maxBranchSessionSize = 16384
    # globe session size , if exceeded throws exceptions
    maxGlobalSessionSize = 512
    # file buffer size , if exceeded allocate new buffer
    fileWriteBufferCacheSize = 16384
    # when recover batch read size
    sessionReloadReadSize = 100
    # async, sync
    flushDiskMode = async
  }
}
```

##### 2.registry.conf

registry{}中是注册中心相关配置，config{}中是配置中心相关配置。seata中，注册中心和配置中心是分开实现的，是两个东西。

我们这里用eureka作注册中心，所以，只用修改registry{}中的：
```java
## **原本配置**：
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "eureka"  修改这里，指明注册中心使用什么

  nacos {
    serverAddr = "localhost"
    namespace = ""
    cluster = "default"
  }
  eureka {
    serviceUrl = "http://localhost:8761/eureka"  修改这里注册中心地址
    application = "default"  
    weight = "1"
  }
  redis {
    serverAddr = "localhost:6379"
    db = "0"
  }
  zk {
    cluster = "default"
    serverAddr = "127.0.0.1:2181"
    session.timeout = 6000
    connect.timeout = 2000
  }
  consul {
    cluster = "default"
    serverAddr = "127.0.0.1:8500"
  }
  etcd3 {
    cluster = "default"
    serverAddr = "http://localhost:2379"
  }
  sofa {
    serverAddr = "127.0.0.1:9603"
    application = "default"
    region = "DEFAULT_ZONE"
    datacenter = "DefaultDataCenter"
    cluster = "default"
    group = "SEATA_GROUP"
    addressWaitTime = "3000"
  }
  file {
    name = "file.conf"
  }
}
```
其他的配置可以暂时使用默认值。

-------
```java
## **本demo服务端registry.conf配置**：
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "eureka"
  eureka {
    serviceUrl = "http://localhost:8761/eureka"
    application = "default"
    weight = "1"
  }
}

config {
  # file、nacos 、apollo、zk、consul、etcd3
  type = "file"

  file {
    name = "file.conf"
  }
}
```

现在已经完成配置修改了，等eureka启动后，就可以启动seata-server了：
如果是在windows下启动seata-server：执行/bin/seata-server.bat即可；
如果是在mac/linux下启动seata-server：sh /bin/seata-server.sh即可；

### 5.client本Demo（自己项目）端相关配置
#### 1.普通配置
client端的几个服务，都是普通的springboot整合了springCloud组件的正常服务，所以，你需要配置eureka，数据库，mapper扫描等，即使不使用seata，你也需要做，这里不做特殊说明，看代码就好。

#### 2.特殊配置
##### 1.application.yml
以order服务为例，除了常规配置外，这里还要配置下事务组信息：（每个需要分布式的项目中都需要这个，所以都需要配置）
```java
spring:
    application:
        name: order-server
    cloud:
        alibaba:
            seata:
                tx-service-group: fsp_tx_group  
##这个fsp_tx_group自定义命名很重要，server，client都要保持一致
```
##### 2.file.conf
**注意：file.conf、registry.conf每个需要分布式事务的项目中都要有这两个文件，修改好之后直接复制就可以**
主要是修改下面：fsp_tx_group 这个名字为自己改的那个名字，要保持统一；
```java
transport {
  # tcp udt unix-domain-socket
  type = "TCP"
  #NIO NATIVE
  server = "NIO"
  #enable heartbeat
  heartbeat = true
  # the client batch send request enable
  enableClientBatchSendRequest = true
  #thread factory for netty
  threadFactory {
    bossThreadPrefix = "NettyBoss"
    workerThreadPrefix = "NettyServerNIOWorker"
    serverExecutorThread-prefix = "NettyServerBizHandler"
    shareBossWorker = false
    clientSelectorThreadPrefix = "NettyClientSelector"
    clientSelectorThreadSize = 1
    clientWorkerThreadPrefix = "NettyClientWorkerThread"
    # netty boss thread size,will not be used for UDT
    bossThreadSize = 1
    #auto default pin or 8
    workerThreadSize = "default"
  }
  shutdown {
    # when destroy server, wait seconds
    wait = 3
  }
  serialization = "seata"
  compressor = "none"
}
service {
  #transaction service group mapping
  
  ########注要是修改fsp_tx_group这个名字############
  vgroupMapping.fsp_tx_group = "default"
  #only support when registry.type=file, please don't set multiple addresses
  default.grouplist = "127.0.0.1:8091"
  #degrade, current not support
  enableDegrade = false
  #disable seata
  disableGlobalTransaction = false
}

client {
  rm {
    asyncCommitBufferLimit = 10000
    lock {
      retryInterval = 10
      retryTimes = 30
      retryPolicyBranchRollbackOnConflict = true
    }
    reportRetryCount = 5
    tableMetaCheckEnable = false
    reportSuccessEnable = false
  }
  tm {
    commitRetryCount = 5
    rollbackRetryCount = 5
  }
  undo {
    dataValidation = true
    logSerialization = "jackson"
    logTable = "undo_log"
  }
  log {
    exceptionRate = 100
  }
}
```
##### 3.registry.conf
使用eureka做注册中心，仅需要修改eureka的配置即可：
```java
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "eureka"

  eureka {
    serviceUrl = "http://localhost:8761/eureka"
    application = "default"
    weight = "1"
  }
}

config {
  # file、nacos 、apollo、zk、consul、etcd3、springCloudConfig
  type = "file"

  file {
    name = "file.conf"
  }
}
```
其他的使用默认值就好。

#### 3.数据源代理
这个是要特别注意的地方，seata对数据源做了代理和接管，**在每个参与分布式事务的服务中，都要做如下配置：**
```java
/**
 * 数据源代理
 * @author wangzhongxiang
 */
@Configuration
public class DataSourceConfiguration {

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource")
    public DataSource druidDataSource(){
        DruidDataSource druidDataSource = new DruidDataSource();
        return druidDataSource;
    }

    @Primary
    @Bean("dataSource")
    public DataSourceProxy dataSource(DataSource druidDataSource){
        return new DataSourceProxy(druidDataSource);
    }

    @Bean
    public SqlSessionFactory sqlSessionFactory(DataSourceProxy dataSourceProxy)throws Exception{
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(dataSourceProxy);
        sqlSessionFactoryBean.setMapperLocations(new PathMatchingResourcePatternResolver()
        .getResources("classpath*:/mapper/*.xml"));
        sqlSessionFactoryBean.setTransactionFactory(new SpringManagedTransactionFactory());
        return sqlSessionFactoryBean.getObject();
    }

}
```
以及：每个分布式事务的启动类都需要加注解（@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)）：比如
```java
/**
 * 订单服务
 * @author wangzhongxiang
 */
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
@MapperScan("io.seata.sample.dao")
@EnableDiscoveryClient
@EnableFeignClients
public class OrderServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(OrderServerApplication.class, args);
	}
}
```
### 6.启动测试
- 1.启动eureka;注册中心
- 2.启动seata-server;服务端
- 3.启动order,storage,account服务;自己的项目
- 4.访问：http://localhost:8180/order/create?userId=1&productId=1&count=10&money=100

然后可以模拟正常情况，异常情况，超时情况等，观察数据库即可。

这个demo,未做各种优化，如果压测，需要修改和优化一些配置，压测出错了，不一定是seata的锅，自己先排查，再去群里问问。

### 7.日志
正常情况：
##### 1.order
```java
2019-09-06 15:44:33.536  INFO 53904 --- [io-8080-exec-10] i.seata.tm.api.DefaultGlobalTransaction  : Begin new global transaction [192.168.158.133:8091:2021468859]
2019-09-06 15:44:33.536  INFO 53904 --- [io-8080-exec-10] c.j.order.service.OrderServiceImpl       : ------->交易开始
2019-09-06 15:44:34.376  INFO 53904 --- [io-8080-exec-10] c.j.order.service.OrderServiceImpl       : ------->交易结束
2019-09-06 15:44:34.593  INFO 53904 --- [io-8080-exec-10] i.seata.tm.api.DefaultGlobalTransaction  : [192.168.158.133:8091:2021468859] commit status:Committed
2019-09-06 15:44:35.296  INFO 53904 --- [atch_RMROLE_6_8] i.s.core.rpc.netty.RmMessageListener     : onMessage:xid=192.168.158.133:8091:2021468859,branchId=2021468861,branchType=AT,resourceId=jdbc:mysql://116.62.62.26/seat-order,applicationData=null
2019-09-06 15:44:35.297  INFO 53904 --- [atch_RMROLE_6_8] io.seata.rm.AbstractRMHandler            : Branch committing: 192.168.158.133:8091:2021468859 2021468861 jdbc:mysql://116.62.62.26/seat-order null
2019-09-06 15:44:35.297  INFO 53904 --- [atch_RMROLE_6_8] io.seata.rm.AbstractRMHandler            : Branch commit result: PhaseTwo_Committed
```
##### 2.storage
```java
2019-09-06 15:44:33.776  INFO 9704 --- [nio-8082-exec-1] c.j.storage.service.StorageServiceImpl   : ------->扣减库存开始
2019-09-06 15:44:34.030  INFO 9704 --- [nio-8082-exec-1] c.j.storage.service.StorageServiceImpl   : ------->扣减库存结束
2019-09-06 15:44:35.422  INFO 9704 --- [atch_RMROLE_5_8] i.s.core.rpc.netty.RmMessageListener     : onMessage:xid=192.168.158.133:8091:2021468859,branchId=2021468864,branchType=AT,resourceId=jdbc:mysql://116.62.62.26/seat-storage,applicationData=null
2019-09-06 15:44:35.423  INFO 9704 --- [atch_RMROLE_5_8] io.seata.rm.AbstractRMHandler            : Branch committing: 192.168.158.133:8091:2021468859 2021468864 jdbc:mysql://116.62.62.26/seat-storage null
2019-09-06 15:44:35.423  INFO 9704 --- [atch_RMROLE_5_8] io.seata.rm.AbstractRMHandler            : Branch commit result: PhaseTwo_Committed
```

##### 3.account
```java
2019-09-06 15:44:34.039  INFO 36556 --- [nio-8081-exec-5] c.j.account.service.AccountServiceImpl   : ------->扣减账户开始
2019-09-06 15:44:34.039  INFO 36556 --- [nio-8081-exec-5] c.j.account.service.AccountServiceImpl   : ------->扣减账户结束
2019-09-06 15:44:35.545  INFO 36556 --- [atch_RMROLE_3_8] i.s.core.rpc.netty.RmMessageListener     : onMessage:xid=192.168.158.133:8091:2021468859,branchId=2021468867,branchType=AT,resourceId=jdbc:mysql://116.62.62.26/seat-account,applicationData=null
2019-09-06 15:44:35.545  INFO 36556 --- [atch_RMROLE_3_8] io.seata.rm.AbstractRMHandler            : Branch committing: 192.168.158.133:8091:2021468859 2021468867 jdbc:mysql://116.62.62.26/seat-account null
2019-09-06 15:44:35.545  INFO 36556 --- [atch_RMROLE_3_8] io.seata.rm.AbstractRMHandler            : Branch commit result: PhaseTwo_Committed
```
### 8.模拟异常
在AccountServiceImpl中模拟异常情况，然后可以查看日志
```java
    /**
     * 扣减账户余额
     * @param userId 用户id
     * @param money 金额
     */
    @Override
    public void decrease(Long userId, BigDecimal money) {
        LOGGER.info("------->扣减账户开始");
        LOGGER.info("------->Account 模拟异常");
        int i = 1 / 0;
        LOGGER.info("------->扣减账户结束");
        accountDao.decrease(userId,money);
    }
```
### 9.调用成环
前面的调用链为order->storage->account;
这里测试的成环是指order->storage->account->order，
这里的account服务又会回头去修改order在前面添加的数据。
经过测试，是支持此种场景的。
```java
    /**
     * 扣减账户余额
     * @param userId 用户id
     * @param money 金额
     */
    @Override
    public void decrease(Long userId, BigDecimal money) {
        LOGGER.info("------->扣减账户开始account中");
        //模拟超时异常，全局事务回滚
//        try {
//            Thread.sleep(30*1000);
//        } catch (InterruptedException e) {
//            e.printStackTrace();
//        }
        accountDao.decrease(userId,money);
        LOGGER.info("------->扣减账户结束account中");

        //修改订单状态，此调用会导致调用成环
        LOGGER.info("修改订单状态开始");
        String mes = orderApi.update(userId, money.multiply(new BigDecimal("0.09")),0);
        LOGGER.info("修改订单状态结束：{}",mes);
    }
```
在最初的order会创建一个订单，然后扣减库存，然后扣减账户，账户扣减完，会回头修改订单的金额和状态，这样调用就成环了。

### 10.seata-server服务端 HA
部署集群，第一台和第二台配置相同，在server端的registry.conf中，注意：
```java
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "eureka"
...... 
  eureka {
    serviceUrl = "http://192.168.xx.xx:8761/eureka"  //两台tcc相同,注册中心的地址
    application = "default" //两台tc相同
    weight = "1"  //权重，截至0.9版本，暂时不支持此参数
  }
 ......
```
注意上述配置和client的配置要一致，2台和多台情况相同。

0.9及之前版本，多tc时，tc会误报异常，此问题0.9之后已经修复，之后的版本应该不会出现此问题。