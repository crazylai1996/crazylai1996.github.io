---
title: 分布式事务 -两阶段提交及Atomikos在Spring Boot的使用
date: 2023-05-01 18:23:31
tags: 分布式事务
---


## 基于XA规范的两阶段提交方式
事务在业务的开发中有着至关重要的作用，事务具有的ACID的特性能保证业务处理前后数据的一致性：
**原子性（Atomicity）：** 事务执行的所有操作，要么全部执行，要么全部不执行；
**一致性（Consistency）：** 事务的执行前后，数据的完整性保持一致；
**隔离性（Isolation）：** 两个或多个事务并行执行时是互不干扰的；
**持久性（Durability）：** 事务执行完成后，其对数据库数据的更改会被永久保存下来；
在单机环境下，数据库系统对事务的支持是比较完善的；但当对数据进行水平或垂直拆分，一个数据库节点变为多个数据库节点时，分布式事务就出现了。

<!--more-->

### XA规范
XA是X/Open组织提出的一个分布式事务的规范，其定义了一个分布式事务的处理模型——DTP。在DTP中定义了三个组件：
Application Program（AP）：应用程序，即业务层，它定义了事务的边界，以及构成该事务的特定操作；
Resource Manager（RM）：资源管理器，可以理解为一个DBMS系统，或者消息服务器管理系统；
Transaction Manager（TM）：事务管理器，也称为协调者，负责协调和管理事务；

AP与RM之间，AP通过RM提供的API进行交互，当需要进行分布式事务时，则向TM发起一个全局事务，TM与RM之间则通过XA接口进行交互，TM管理了到RM的链接，并实现了两阶段提交。

### 两阶段提交流程（2PC）
XA规范中，多个RM状态之间的协调通过TM进行，而这个资源协调的过程采用了两阶段提交协议（2PC），2PC实际上是一种在多节点之间实现事务原子提交的算法，它用来确保所有节点要么全部提交，要么全部中止。

在2PC中，分为准备阶段和提交阶段：
第一阶段：发送一个准备请求到所有参与者节点，询问他们是否可以提交；

第二阶段：如果所有参与者节点回答“是”，则表示他们已准备好提交，那么协调者将在阶段2发出提交请求；

![1p](http://storage.laixiaoming.space/blog/1p.jpg)

![2p](http://storage.laixiaoming.space/blog/2p.jpg)

如果在准备阶段，有一个RM返回失败时，则在第二个阶段将回滚所有资源

![1pc-error](http://storage.laixiaoming.space/blog/1pc-error.jpg)

![2pc-error](http://storage.laixiaoming.space/blog/2pc-error.jpg)



### 2PC的局限性

2PC能基本满足了事务的 ACID 特性，但也存在着明显的缺点：

 - 在事务的执行过程中，所有的参与节点都是阻塞型的，在并发量高的系统中，性能受限严重；
 - 如果TM在commit前发生故障，那么所有参与节点会因为无法提交事务而处于长时间锁定资源的状态；
  - 在实际情况中，由于分布式环境下的复杂性，TM在发送commit请求后，可能因为局部网络原因，导致只有部分参与者收到commit请求时，系统便出现了数据不一致的现象；
  - XA协议要求所有参与者需要与TM进行直接交互，但在微服务架构下，一个服务与多个RM直接关联常常是被不允许的；



## Atomikos在Spring Boot的使用
Atomikos在XA中作为一个事务管理器（TM）存在。在Spring Boot应用中，可以通过Atomikos在应用中方便的引入分布式事务。
下面以一个简单的订单创建流程的为例：
### 引入依赖
```
<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jta-atomikos</artifactId>
        </dependency>
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>2.1.2</version>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.11</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
            <version>1.1.22</version>
        </dependency>
```

### 配置数据源
application.yml
```
spring:
  datasource:
    druid:
      order-db:
        name: order-db
        url: jdbc:mysql://localhost:3307/order?useSSL=false&serverTimezone=Asia/Shanghai
        username: root
        password: mysql
      product-db:
        name: order-db
        url: jdbc:mysql://localhost:3306/product?useSSL=false&serverTimezone=Asia/Shanghai
        username: root
        password: mysql
  jta:
    transaction-manager-id: order-product-tx-manager
```
```
@Configuration
@MapperScan(basePackages = "gdou.laixiaoming.atomikos.demo.mapper.order", sqlSessionFactoryRef = "orderSqlSessionFactory")
public class OrderDataSourceConfig {

    @Bean(name = "druidOrderDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.druid.order-db")
    public DruidXADataSource druidOrderDataSource(){
        DruidXADataSource xaDataSource = new DruidXADataSource();
        return xaDataSource;
    }

    @Bean(name = "orderDataSource")
    public DataSource orderDataSource(
            @Qualifier("druidOrderDataSource") DruidXADataSource druidOrderDataSource) {
        AtomikosDataSourceBean ds = new AtomikosDataSourceBean();
        ds.setXaDataSource(druidOrderDataSource);
        ds.setXaDataSourceClassName("com.alibaba.druid.pool.xa.DruidXADataSource");
        ds.setUniqueResourceName("orderDataSource");
        return ds;
    }

    @Bean
    public SqlSessionFactory orderSqlSessionFactory(
            @Qualifier("orderDataSource") DataSource orderDataSource) throws Exception {
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(orderDataSource);
        return sqlSessionFactoryBean.getObject();
    }
}
```
```
@Configuration
@MapperScan(basePackages = "gdou.laixiaoming.atomikos.demo.mapper.product", sqlSessionFactoryRef = "productSqlSessionFactory")
public class ProductDataSourceConfig {

    @Bean(name = "druidProductDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.druid.product-db")
    public DruidXADataSource druidProductDataSource(){
        DruidXADataSource xaDataSource = new DruidXADataSource();
        return xaDataSource;
    }

    @Bean(name = "productDataSource")
    public DataSource productDataSource(
            @Qualifier("druidProductDataSource") DruidXADataSource druidProductDataSource) {
        AtomikosDataSourceBean ds = new AtomikosDataSourceBean();
        ds.setXaDataSource(druidProductDataSource);
        ds.setXaDataSourceClassName("com.alibaba.druid.pool.xa.DruidXADataSource");
        ds.setUniqueResourceName("productDataSource");
        return ds;
    }

    @Bean
    public SqlSessionFactory productSqlSessionFactory(
            @Qualifier("productDataSource") DataSource productDataSource) throws Exception {
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(productDataSource);
        return sqlSessionFactoryBean.getObject();
    }
}

```

### 构建商品服务
```
@Service
public class ProductServiceImpl implements ProductService {

    @Autowired
    private ProductMapper productMapper;

    @Override
    public void updateInventory(Long productId) {
    	//模拟异常流程
        if(productId == 2){
            throw new RuntimeException("更新库存失败");
        }
        productMapper.updateInventory(productId);
    }
}
```

### 构建订单服务
```
@Service
public class OrderServiceImpl implements OrderService {

    @Autowired
    private OrderMapper orderMapper;
    @Autowired
    private ProductService productService;

    @Transactional(rollbackFor = RuntimeException.class)
    @Override
    public void order(Long productId) {
        orderMapper.addOrder(productId);
        productService.updateInventory(productId);
    }
}
```

### 测试
```
@SpringBootTest
@RunWith(SpringRunner.class)
public class ServiceTest {

    @Autowired
    private OrderService orderService;

    @Test
    public void testCommit() {
        orderService.order(1L);
    }

    @Test
    public void testRollback() {
        orderService.order(2L);
    }
    
}
```
通过运行测试用例，我们可以发现testCommit()方法在订单库以及商品库的成功完成的修改；而testRollback()方法则因为商品服务异常进行了回滚，回滚后的订单库和商品库数据都恢复到了事务开启前的状态。

参考：
《大型网站系统与Java中间件实践》
[SpringBoot Atomikos 多数据源分布式事务](https://www.jianshu.com/p/f9bac5822d30)
