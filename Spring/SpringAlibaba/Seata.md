[官方文档](https://seata.io/zh-cn/docs/overview/what-is-seata.html)
在分布式环境下实现事务

在父工程引入了cloud-alibaba相关依赖管理后，在微服务中引入
依赖：
```xml
<dependency>  
    <groupId>com.alibaba.cloud</groupId>  
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>  
</dependency>
```
使用：
添加file.conf

在入口service中打上注解`@GlobalTransactional`

### 分布式事务解决方案

要开始实现分布式事务，我们得先从理论上开始下手，我们来了解一下常用的分布式事务解决方案。

1. **XA分布式事务协议 - 2PC（两阶段提交实现）**
    
    这里的PC实际上指的是Prepare和Commit，也就是说它分为两个阶段，一个是准备一个是提交，整个过程的参与者一共有两个角色，一个是事务的执行者，一个是事务的协调者，实际上整个分布式事务的运作都需要依靠协调者来维持：
    
    ![image-20230306233211675](https://image.itbaima.cn/markdown/2023/03/06/BWiUzFrjHAao1kJ.png)
    
    在准备和提交阶段，会进行：
    
    - **准备阶段：**
        
        一个分布式事务是由协调者来开启的，首先协调者会向所有的事务执行者发送事务内容，等待所有的事务执行者答复。
        
        各个事务执行者开始执行事务操作，但是不进行提交，并将undo和redo信息记录到事务日志中。
        
        如果事务执行者执行事务成功，那么就告诉协调者成功Yes，否则告诉协调者失败No，不能提交事务。
        
    - **提交阶段：**
        
        当所有的执行者都反馈完成之后，进入第二阶段。
        
        协调者会检查各个执行者的反馈内容，如果所有的执行者都返回成功，那么就告诉所有的执行者可以提交事务了，最后再释放锁资源。
        
        如果有至少一个执行者返回失败或是超时，那么就让所有的执行者都回滚，分布式事务执行失败。
        
    
    虽然这种方式看起来比较简单，但是存在以下几个问题：
    
    - 事务协调者是非常核心的角色，一旦出现问题，将导致整个分布式事务不能正常运行。
    - 如果提交阶段发生网络问题，导致某些事务执行者没有收到协调者发来的提交命令，将导致某些执行者提交某些执行者没提交，这样肯定是不行的。
2. **XA分布式事务协议 - 3PC（三阶段提交实现）**
    
    三阶段提交是在二阶段提交基础上的改进版本，主要是加入了超时机制，同时在协调者和执行者中都引入了超时机制。
    
    三个阶段分别进行：
    
    - **CanCommit阶段：**
        
        协调者向执行者发送CanCommit请求，询问是否可以执行事务提交操作，然后开始等待执行者的响应。
        
        执行者接收到请求之后，正常情况下，如果其自身认为可以顺利执行事务，则返回Yes响应，并进入预备状态，否则返回No
        
    - **PreCommit阶段：**
        
        协调者根据执行者的反应情况来决定是否可以进入第二阶段事务的PreCommit操作。
        
        如果所有的执行者都返回Yes，则协调者向所有执行者发送PreCommit请求，并进入Prepared阶段，执行者接收到请求后，会执行事务操作，并将undo和redo信息记录到事务日志中，如果成功执行，则返回成功响应。
        
        如果所有的执行者至少有一个返回No，则协调者向所有执行者发送abort请求，所有的执行者在收到请求或是超过一段时间没有收到任何请求时，会直接中断事务。
        
    - **DoCommit阶段：**
        
        该阶段进行真正的事务提交。
        
        协调者接收到所有执行者发送的成功响应，那么他将从PreCommit状态进入到DoCommit状态，并向所有执行者发送doCommit请求，执行者接收到doCommit请求之后，开始执行事务提交，并在完成事务提交之后释放所有事务资源，并最后向协调者发送确认响应，协调者接收到所有执行者的确认响应之后，完成事务（如果因为网络问题导致执行者没有收到doCommit请求，执行者会在超时之后直接提交事务，虽然执行者只是猜测协调者返回的是doCommit请求，但是因为前面的两个流程都正常执行，所以能够在一定程度上认为本次事务是成功的，因此会直接提交）
        
        协调者没有接收至少一个执行者发送的成功响应（也可能是响应超时），那么就会执行中断事务，协调者会向所有执行者发送abort请求，执行者接收到abort请求之后，利用其在PreCommit阶段记录的undo信息来执行事务的回滚操作，并在完成回滚之后释放所有的事务资源，执行者完成事务回滚之后，向协调者发送确认消息， 协调者接收到参与者反馈的确认消息之后，执行事务的中断。
        
    
    相比两阶段提交，三阶段提交的优势是显而易见的，当然也有缺点：
    
    - 3PC在2PC的第一阶段和第二阶段中插入一个准备阶段，保证了在最后提交阶段之前各参与节点的状态是一致的。
    - 一旦参与者无法及时收到来自协调者的信息之后，会默认执行Commit，这样就不会因为协调者单方面的故障导致全局出现问题。
    - 但是我们知道，实际上超时之后的Commit决策本质上就是一个赌注罢了，如果此时协调者发送的是abort请求但是超时未接收，那么就会直接导致数据一致性问题。
3. **TCC（补偿事务）**
    
    补偿事务TCC就是Try、Confirm、Cancel，它对业务有侵入性，一共分为三个阶段，我们依次来解读一下。
    
    - **Try阶段：**
        
        比如我们需要在借书时，将书籍的库存`-1`，并且用户的借阅量也`-1`，但是这个操作，除了直接对库存和借阅量进行修改之外，还需要将减去的值，单独存放到冻结表中，但是此时不会创建借阅信息，也就是说只是预先把关键的东西给处理了，预留业务资源出来。
        
    - **Confirm阶段：**
        
        如果Try执行成功无误，那么就进入到Confirm阶段，接着之前，我们就该创建借阅信息了，只能使用Try阶段预留的业务资源，如果创建成功，那么就对Try阶段冻结的值，进行解冻，整个流程就完成了。当然，如果失败了，那么进入到Cancel阶段。
        
    - **Cancel阶段：**
        
        不用猜了，那肯定是把冻结的东西还给人家，因为整个借阅操作压根就没成功。就像你付了款买了东西但是网络问题，导致交易失败，钱不可能不还给你吧。
        
    
    跟XA协议相比，TCC就没有协调者这一角色的参与了，而是自主通过上一阶段的执行情况来确保正常，充分利用了集群的优势，性能也是有很大的提升。但是缺点也很明显，它与业务具有一定的关联性，需要开发者去编写更多的补偿代码，同时并不一定所有的业务流程都适用于这种形式。

### Seata机制简介

前面我们了解了一些分布式事务的解决方案，那么我们来看一下Seata是如何进行分布式事务的处理的。

![image-20230306233227905](https://image.itbaima.cn/markdown/2023/03/06/LsUq3AvrfhQJPCz.png)

官网给出的是这样的一个架构图，那么图中的RM、TM、TC代表着什么意思呢？

- RM（Resource Manager）：用于直接执行本地事务的提交和回滚。
- TM（Transaction Manager）：TM是分布式事务的核心管理者。比如现在我们需要在借阅服务中开启全局事务，来让其自身、图书服务、用户服务都参与进来，也就是说一般全局事务发起者就是TM。
- TC（Transaction Manager）这个就是我们的Seata服务器，用于全局控制，比如在XA模式下就是一个协调者的角色，而一个分布式事务的启动就是由TM向TC发起请求，TC再来与其他的RM进行协调操作。

> TM请求TC开启一个全局事务，TC会生成一个XID作为该全局事务的编号，XID会在微服务的调用链路中传播，保证将多个微服务的子事务关联在一起；RM请求TC将本地事务注册为全局事务的分支事务，通过全局事务的XID进行关联；TM请求TC告诉XID对应的全局事务是进行提交还是回滚；TC驱动RM将XID对应的自己的本地事务进行提交还是回滚；

Seata支持4种事务模式，官网文档：https://seata.io/zh-cn/docs/overview/what-is-seata.html

- AT：本质上就是2PC的升级版，在 AT 模式下，用户只需关心自己的 “业务SQL”
    
    1. 一阶段，Seata 会拦截“业务 SQL”，首先解析 SQL 语义，找到“业务 SQL”要更新的业务数据，在业务数据被更新前，将其保存成“before image”，然后执行“业务 SQL”更新业务数据，在业务数据更新之后，再将其保存成“after image”，最后生成行锁。以上操作全部在一个数据库事务内完成，这样保证了一阶段操作的原子性。
    2. 二阶段如果确认提交的话，因为“业务 SQL”在一阶段已经提交至数据库， 所以 Seata 框架只需将一阶段保存的快照数据和行锁删掉，完成数据清理即可，当然如果需要回滚，那么就用“before image”还原业务数据；但在还原前要首先要校验脏写，对比“数据库当前业务数据”和 “after image”，如果两份数据完全一致就说明没有脏写，可以还原业务数据，如果不一致就说明有脏写，出现脏写就需要转人工处理。
- TCC：和我们上面讲解的思路是一样的。
    
- XA：同上，但是要求数据库本身支持这种模式才可以。
    
- Saga：用于处理长事务，每个执行者需要实现事务的正向操作和补偿操作：
    
    ![image-20230306233328901](https://image.itbaima.cn/markdown/2023/03/06/vLTZFS4yn26uabA.png)
    

那么，以AT模式为例，我们的程序如何才能做到不对业务进行侵入的情况下实现分布式事务呢？实际上，Seata客户端，是通过对数据源进行代理实现的，使用的是DataSourceProxy类，所以在程序这边，我们只需要将对应的代理类注册为Bean即可（0.9版本之后支持自动进行代理，不用我们手动操作）

接下来，我们就以AT模式为例进行讲解。

### 使用file模式部署

Seata也是以服务端形式进行部署的，然后每个服务都是客户端，服务端下载地址：https://github.com/seata/seata/releases/download/v1.4.2/seata-server-1.4.2.zip

把源码也下载一下：https://github.com/seata/seata/archive/refs/heads/develop.zip

下载完成之后，放入到IDEA项目目录中，添加启动配置，这里端口使用8868：

![image-20230306233336803](https://image.itbaima.cn/markdown/2023/03/06/ykH1BSPcxlvY4on.png)

Seata服务端支持本地部署或是基于注册发现中心部署（比如Nacos、Eureka等），这里我们首先演示一下最简单的本地部署，不需要对Seata的配置文件做任何修改。

Seata存在着事务分组机制：

- 事务分组：seata的资源逻辑，可以按微服务的需要，在应用程序（客户端）对自行定义事务分组，每组取一个名字。
- 集群：seata-server服务端一个或多个节点组成的集群cluster。 应用程序（客户端）使用时需要指定事务逻辑分组与Seata服务端集群（默认为default）的映射关系。

为啥要设计成通过事务分组再直接映射到集群？干嘛不直接指定集群呢？获取事务分组到映射集群的配置。这样设计后，事务分组可以作为资源的逻辑隔离单位，出现某集群故障时可以快速failover，只切换对应分组，可以把故障缩减到服务级别，但前提也是你有足够server集群。

接着我们需要将我们的各个服务作为Seate的客户端，只需要导入依赖即可：

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
</dependency>
```

然后添加配置：

```yaml
seata:
  service:
    vgroup-mapping:
    	# 这里需要对事务组做映射，默认的分组名为 应用名称-seata-service-group，将其映射到default集群
    	# 这个很关键，一定要配置对，不然会找不到服务
      bookservice-seata-service-group: default
    grouplist:
      default: localhost:8868
```

这样就可以直接启动了，但是注意现在只是单纯地连接上，并没有开启任何的分布式事务。

现在我们接着来配置开启分布式事务，首先在启动类添加注解，此注解会添加一个后置处理器将数据源封装为支持分布式事务的代理数据源（虽然官方表示配置文件中已经默认开启了自动代理，但是实测1.4.2版本下只能打注解的方式才能生效）：

```java
@EnableAutoDataSourceProxy
@SpringBootApplication
public class BookApplication {
    public static void main(String[] args) {
        SpringApplication.run(BookApplication.class, args);
    }
}
```

接着我们需要在开启分布式事务的方法上添加`@GlobalTransactional`注解：

```java
@GlobalTransactional
@Override
public boolean doBorrow(int uid, int bid) {
  	//这里打印一下XID看看，其他的服务业添加这样一个打印，如果一会都打印的是同一个XID，表示使用的就是同一个事务
    System.out.println(RootContext.getXID());
    if(bookClient.bookRemain(bid) < 1)
        throw new RuntimeException("图书数量不足");
    if(userClient.userRemain(uid) < 1)
        throw new RuntimeException("用户借阅量不足");
    if(!bookClient.bookBorrow(bid))
        throw new RuntimeException("在借阅图书时出现错误！");
    if(mapper.getBorrow(uid, bid) != null)
        throw new RuntimeException("此书籍已经被此用户借阅了！");
    if(mapper.addBorrow(uid, bid) <= 0)
        throw new RuntimeException("在录入借阅信息时出现错误！");
    if(!userClient.userBorrow(uid))
        throw new RuntimeException("在借阅时出现错误！");
    return true;
}
```

还没结束，我们前面说了，Seata会分析修改数据的sql，同时生成对应的反向回滚SQL，这个回滚记录会存放在undo_log 表中。所以要求每一个Client 都有一个对应的undo_log表（也就是说每个服务连接的数据库都需要创建这样一个表，这里由于我们三个服务都用的同一个数据库，所以说就只用在这个数据库中创建undo_log表即可），表SQL定义如下：

```sql
CREATE TABLE `undo_log`
(
  `id`            BIGINT(20)   NOT NULL AUTO_INCREMENT,
  `branch_id`     BIGINT(20)   NOT NULL,
  `xid`           VARCHAR(100) NOT NULL,
  `context`       VARCHAR(128) NOT NULL,
  `rollback_info` LONGBLOB     NOT NULL,
  `log_status`    INT(11)      NOT NULL,
  `log_created`   DATETIME     NOT NULL,
  `log_modified`  DATETIME     NOT NULL,
  `ext`           VARCHAR(100) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_undo_log` (`xid`, `branch_id`)
) ENGINE = InnoDB
  AUTO_INCREMENT = 1
  DEFAULT CHARSET = utf8;
```

创建完成之后，我们现在就可以启动三个服务了，我们来测试一下当出现异常的时候是不是会正常回滚：

![image-20230306233351571](https://image.itbaima.cn/markdown/2023/03/06/NIe9QFW3jf1DdnV.png)

 

![image-20230306233359914](https://image.itbaima.cn/markdown/2023/03/06/LwcdO2HuWAhFr5p.png)

首先第一次肯定是正常完成借阅操作的，接着我们再次进行请求，肯定会出现异常：

![image-20230306233408870](https://image.itbaima.cn/markdown/2023/03/06/6VOfsp9UxYJzgKD.png)

 

![image-20230306233417576](https://image.itbaima.cn/markdown/2023/03/06/yEQa2qeiNc5npV9.png)

如果能在栈追踪信息中看到seata相关的包，那么说明分布式事务已经开始工作了，通过日志我们可以看到，出现了回滚操作：

![image-20230306233428386](https://image.itbaima.cn/markdown/2023/03/06/VtBlx4U1TzcqKra.png)

并且数据库中确实是回滚了扣除操作：

![image-20230306233436382](https://image.itbaima.cn/markdown/2023/03/06/WXn9UPgxBVhdHmb.png)

这样，我们就通过Seata简单地实现了分布式事务。

### 使用nacos模式部署

前面我们实现了本地Seata服务的file模式部署，现在我们来看看如何让其配合Nacos进行部署，利用Nacos的配置管理和服务发现机制，Seata能够更好地工作。

我们先单独为Seata配置一个命名空间：

![image-20230306233444767](https://image.itbaima.cn/markdown/2023/03/06/93mXN5dlC2GTLOW.png)

我们打开`conf`目录中的`registry.conf`配置文件：

```properties
registry {
	# 注册配置
	# 可以看到这里可以选择类型，默认情况下是普通的file类型，也就是本地文件的形式进行注册配置
	# 支持的类型如下，对应的类型在下面都有对应的配置
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "nacos"

	# 采用nacos方式会将seata服务端也注册到nacos中，这样客户端就可以利用服务发现自动找到seata服务
	# 就不需要我们手动指定IP和端口了，不过看似方便，坑倒是不少，后面再说
  nacos {
  	# 应用名称，这里默认就行
    application = "seata-server"
    # Nacos服务器地址
    serverAddr = "localhost:8848"
    # 这里使用的是SEATA_GROUP组，一会注册到Nacos中就是这个组
    group = "SEATA_GROUP"
    # 这里就使用我们上面单独为seata配置的命名空间，注意填的是ID
    namespace = "89fc2145-4676-48b8-9edd-29e867879bcb"
    # 集群名称，这里还是使用default
    cluster = "default"
    # Nacos的用户名和密码
    username = "nacos"
    password = "nacos"
  }
  	#...
```

注册信息配置完成之后，接着我们需要将配置文件也放到Nacos中，让Nacos管理配置，这样我们就可以对配置进行热更新了，一旦环境需要变化，只需要直接在Nacos中修改即可。

```properties
config {
	# 这里我们也使用nacos
  # file、nacos 、apollo、zk、consul、etcd3
  type = "nacos"

  nacos {
  	# 跟上面一样的配法
    serverAddr = "127.0.0.1:8848"
    namespace = "89fc2145-4676-48b8-9edd-29e867879bcb"
    group = "SEATA_GROUP"
    username = "nacos"
    password = "nacos"
    # 这个不用改，默认就行
    dataId = "seataServer.properties"
  }
```

接着，我们需要将配置导入到Nacos中，我们打开一开始下载的源码`script/config-center/nacos`目录，这是官方提供的上传脚本，我们直接运行即可（windows下没对应的bat就很蛋疼，可以使用git命令行来运行一下），这里我们使用这个可交互的版本：

![image-20230306233500474](https://image.itbaima.cn/markdown/2023/03/06/1tPwBFn7u3ScCeY.png)

按照提示输入就可以了，不输入就使用的默认值，不知道为啥最新版本有四个因为参数过长还导入失败了，就离谱，不过不影响。

导入成功之后，可以在对应的命名空间下看到对应的配置（为啥非要一个一个配置项单独搞，就不能写一起吗）：

![image-20230306233510973](https://image.itbaima.cn/markdown/2023/03/06/8yTQGZluYVe1cg2.png)

注意，还没完，我们还需要将对应的事务组映射配置也添加上，DataId格式为`service.vgroupMapping.事务组名称`，比如我们就使用默认的名称，值全部依然使用default即可：

![image-20230306233521002](https://image.itbaima.cn/markdown/2023/03/06/UBchb4zPjHAfCSs.png)

现在我们就完成了服务端的Nacos配置，接着我们需要对客户端也进行Nacos配置：

```yaml
seata:
	# 注册
  registry:
  	# 使用Nacos
    type: nacos
    nacos:
    	# 使用Seata的命名空间，这样才能正确找到Seata服务，由于组使用的是SEATA_GROUP，配置默认值就是，就不用配了
      namespace: 89fc2145-4676-48b8-9edd-29e867879bcb
      username: nacos
      password: nacos
  # 配置
  config:
    type: nacos
    nacos:
      namespace: 89fc2145-4676-48b8-9edd-29e867879bcb
      username: nacos
      password: nacos
```

现在我们就可以启动这三个服务了，可以在Nacos中看到Seata以及三个服务都正常注册了：

![image-20230306233529255](https://image.itbaima.cn/markdown/2023/03/06/PSbw5TFhm74Wu3n.png)

 

![image-20230306233538630](https://image.itbaima.cn/markdown/2023/03/06/nZUcuM2kJ86zgBv.png)

接着我们就可以访问一下服务试试看了：

![image-20230306233545257](https://image.itbaima.cn/markdown/2023/03/06/Fn3R2Jrq1YyleCh.png)

可以看到效果和上面是一样的，不过现在我们的注册和配置都继承在Nacos中进行了。

我们还可以配置一下事务会话信息的存储方式，默认是file类型，那么就会在运行目录下创建`file_store`目录，我们可以将其搬到数据库中存储，只需要修改一下配置即可：

![image-20230306233553931](https://image.itbaima.cn/markdown/2023/03/06/Cph9zPF2kaSvKdY.png)

将`store.session.mode`和`store.mode`的值修改为`db`

接着我们对数据库信息进行一下配置：

- 数据库驱动
- 数据库URL
- 数据库用户名密码

其他的默认即可：

![image-20230306233612224](https://image.itbaima.cn/markdown/2023/03/06/dlmYNnARZaxJ5MH.png)

接着我们需要将对应的数据库进行创建，创建seata数据库，然后直接CV以下语句：

```sql
-- -------------------------------- The script used when storeMode is 'db' --------------------------------
-- the table to store GlobalSession data
CREATE TABLE IF NOT EXISTS `global_table`
(
    `xid`                       VARCHAR(128) NOT NULL,
    `transaction_id`            BIGINT,
    `status`                    TINYINT      NOT NULL,
    `application_id`            VARCHAR(32),
    `transaction_service_group` VARCHAR(32),
    `transaction_name`          VARCHAR(128),
    `timeout`                   INT,
    `begin_time`                BIGINT,
    `application_data`          VARCHAR(2000),
    `gmt_create`                DATETIME,
    `gmt_modified`              DATETIME,
    PRIMARY KEY (`xid`),
    KEY `idx_status_gmt_modified` (`status` , `gmt_modified`),
    KEY `idx_transaction_id` (`transaction_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4;

-- the table to store BranchSession data
CREATE TABLE IF NOT EXISTS `branch_table`
(
    `branch_id`         BIGINT       NOT NULL,
    `xid`               VARCHAR(128) NOT NULL,
    `transaction_id`    BIGINT,
    `resource_group_id` VARCHAR(32),
    `resource_id`       VARCHAR(256),
    `branch_type`       VARCHAR(8),
    `status`            TINYINT,
    `client_id`         VARCHAR(64),
    `application_data`  VARCHAR(2000),
    `gmt_create`        DATETIME(6),
    `gmt_modified`      DATETIME(6),
    PRIMARY KEY (`branch_id`),
    KEY `idx_xid` (`xid`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4;

-- the table to store lock data
CREATE TABLE IF NOT EXISTS `lock_table`
(
    `row_key`        VARCHAR(128) NOT NULL,
    `xid`            VARCHAR(128),
    `transaction_id` BIGINT,
    `branch_id`      BIGINT       NOT NULL,
    `resource_id`    VARCHAR(256),
    `table_name`     VARCHAR(32),
    `pk`             VARCHAR(36),
    `status`         TINYINT      NOT NULL DEFAULT '0' COMMENT '0:locked ,1:rollbacking',
    `gmt_create`     DATETIME,
    `gmt_modified`   DATETIME,
    PRIMARY KEY (`row_key`),
    KEY `idx_status` (`status`),
    KEY `idx_branch_id` (`branch_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4;

CREATE TABLE IF NOT EXISTS `distributed_lock`
(
    `lock_key`       CHAR(20) NOT NULL,
    `lock_value`     VARCHAR(20) NOT NULL,
    `expire`         BIGINT,
    primary key (`lock_key`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4;

INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('HandleAllSession', ' ', 0);
-- -------------------------------- storeMode 为 'db' 时使用的脚本 -------------------------------- 
--表中存储 GlobalSession 数据 如果不存在，则创建表 'global_table' （ 'xid' VARCHAR（128） 不为空， 'transaction_id' BIGINT， '状态' TINYINT NOT NULL， 'application_id' VARCHAR（32）， 'transaction_service_group' VARCHAR（32）， 'transaction_name' VARCHAR（128）， 'timeout'                  INT， 'begin_time' BIGINT， 'application_data' VARCHAR（2000）， 'gmt_create' DATETIME， 'gmt_modified' DATETIME， 主键 （'xid'）， 键 'idx_status_gmt_modified' （'status' ， 'gmt_modified'）， 键 'idx_transaction_id' （'transaction_id'） ） ENGINE = InnoDB 默认字符集 = utf8mb4; -- 用于存储 BranchSession 数据的表 如果不存在，则创建表 'branch_table' （ 'branch_id' BIGINT 不为 NULL， 'xid' VARCHAR（128） 不为 NULL， 'transaction_id' BIGINT， 'resource_group_id' VARCHAR（32）， 'resource_id' VARCHAR（256）， 'branch_type' VARCHAR（8）， '状态' TINYINT， 'client_id' VARCHAR（64）， 'application_data' VARCHAR（2000）， 'gmt_create' 日期时间 （6）， 'gmt_modified'     DATETIME（6）， 主键 （'branch_id'）， 键 'idx_xid' （'xid'） ） ENGINE = InnoDB DEFAULT CHARSET = utf8mb4; -- 用于存储锁定数据的表 如果不存在，则创建表 'lock_table' （ 'row_key' VARCHAR（128） 不为空， 'XID' VARCHAR（128）， 'transaction_id' BIGINT， 'branch_id' BIGINT 不为空， 'resource_id' VARCHAR（256）， 'table_name' VARCHAR（32）， 'PK' VARCHAR（36）， 'STATUS' TINYINT NOT NULL DEFAULT '0' 注释 '0：锁定 ，1：回滚'， 'gmt_create' DATETIME， 'gmt_modified' DATETIME，     主键 （'row_key'）， 键 'idx_status' （'status'）， 键 'idx_branch_id' （'branch_id'） ） ENGINE = InnoDB DEFAULT CHARSET = utf8mb4; 如果不存在，则创建表 'distributed_lock' （ 'lock_key' CHAR（20） NOT NULL， 'lock_value' VARCHAR（20） NOT NULL， 'EXPIRE' BIGINT， 主键 （'lock_key'） ） ENGINE = InnoDB 默认字符集 = utf8mb4; INSERT INTO 'distributed_lock' （lock_key， lock_value， expires） VALUES （'HandleAllSession'， ' '， 0）;
```

![image-20230306233627086](https://image.itbaima.cn/markdown/2023/03/06/7zvewSLhFmbc8G1.png)

完成之后，重启Seata服务端即可：

![image-20230306233752098](https://image.itbaima.cn/markdown/2023/03/06/G7qQoEy8DCX9bLJ.png)

看到了数据源初始化成功，现在已经在使用数据库进行会话存储了。

如果Seata服务端出现报错，可能是我们自定义事务组的名称太长了：

![image-20230306233933641](https://image.itbaima.cn/markdown/2023/03/06/qoNhgzM2PXpZU9B.png)

将`globle_table`表的字段`transaction_server_group`长度适当增加一下即可：

![image-20230306233940850](https://image.itbaima.cn/markdown/2023/03/06/9LnaoUxHzlY1GdV.png)

