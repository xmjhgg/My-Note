# Dubbo总结



Dubbo是一款RPC（  **Remote  Procedure  Call**  远程过程调用）框架，可以简便的调用另一台计算机中的某个函数，并提供了被调用端的负载均衡、流量监控、服务自动注册等功能



- Dubbo架构
- Dubbo的开发使用
- Dubbo底层调用过程简要说明





## Dubbo架构

![](D:\documentary\mdImgs\Dubbo\2.jpg)

Dubbo的架构如上

### 角色说明

- Provider（服务提供者）

  远程服务的提供方

- Consumer（服务消费者）

  远程服务的调用方

- Registry（注册中心）

  远程服务的注册中心和订阅中心

- Container（服务容器）

  远程服务提供方的运行时的容器

- Monitor（监控中心）

  远程服务的数据监控

### 角色执行关系

0. Container负责启动，加载，运行服务提供者
1. Provider在启动时，向Registry注册自己提供的服务
2. Consumer在启动时，向Registry订阅自己需要的服务
3. Registry返回服务提供者地址列表给Consumer，如果列表有变动，Registry将会基于长连接推送给Consumer
4. Consumer从提供者地址列表中，基于软负载均衡算法，选一台Provider进行调用，如果调用失败，再选另一台调用。
5. Consumer和Provider会在内存中累计调用次数和调用时间，每分钟发一次统计给Monitor

### 角色连接关系

- 注册中心负责服务地址的注册与查找，服务提供者和消费者只在启动时与注册中心交互，注册中心不转发请求，压力较小，当服务提供者发生变更时，会推送事件给消费者
- 注册中心，服务提供者，服务消费者三者之间均为长连接，监控中心除外

- 注册中心通过长连接感知服务提供者的存在，服务提供者宕机，注册中心将立即推送事件通知消费者

- 注册中心和监控中心全部宕机，不影响已运行的提供者和消费者，消费者在本地缓存了提供者列表

- 注册中心和监控中心都是可选的，服务消费者可以直连服务提供者



## Dubbo的开发使用

这一节简单介绍Dubbo在项目中如何使用

- 安装注册中心

  官方建议使用Zookeeper作为Dubbo的注册中心，并在Dubbo的配置文件中设置注册中心的地址

  ```java
  <dubbo:registry protocol="zookeeper" address="127.0.0.1:2181"></dubbo:registry>
  ```

- 定义服务接口

  建议服务接口单独设置为一个项目，Provider和Consumer一起引用这个项目的接口

  ```java
  public interface DemoService {
      String sayHello(String name);
  }
  ```

- 在服务提供方实现接口

  ```java
  public class DemoServiceImpl implements DemoService {
      public String sayHello(String name) {
          return "Hello " + name;
      }
  }
  
  ```

- 暴露服务（向注册中心注册的配置文件）

  ```java
  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
      xsi:schemaLocation="http://www.springframework.org/schema/beans        http://www.springframework.org/schema/beans/spring-beans-4.3.xsd        http://dubbo.apache.org/schema/dubbo        http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
   
      <!-- 提供方应用信息，用于计算依赖关系 -->
      <dubbo:application name="hello-world-app"  />
   
      <!-- 使用multicast广播注册中心暴露服务地址 -->
      <dubbo:registry address="multicast://224.5.6.7:1234" />
   
      <!-- 用dubbo协议在20880端口暴露服务 -->
      <dubbo:protocol name="dubbo" port="20880" />
   
      <!-- 声明需要暴露的服务接口 -->
      <dubbo:service interface="org.apache.dubbo.demo.DemoService" ref="demoService" />
   
      <!-- 和本地bean一样实现服务 -->
      <bean id="demoService" class="org.apache.dubbo.demo.provider.DemoServiceImpl" />
  </beans>
  ```

- 消费者配置

  ```java
  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
      xsi:schemaLocation="http://www.springframework.org/schema/beans        http://www.springframework.org/schema/beans/spring-beans-4.3.xsd        http://dubbo.apache.org/schema/dubbo        http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
   
      <!-- 消费方应用名，用于计算依赖关系，不是匹配条件，不要与提供方一样 -->
      <dubbo:application name="consumer-of-helloworld-app"  />
   
      <!-- 使用multicast广播注册中心暴露发现服务地址 -->
      <dubbo:registry address="multicast://224.5.6.7:1234" />
   
      <!-- 生成远程服务代理，可以和本地bean一样使用demoService -->
      <dubbo:reference id="demoService" interface="org.apache.dubbo.demo.DemoService" />
  </beans>
  ```

- 调用提供者的方法（和本地Bean一样调用）

  ```java
  public class Consumer {
      public static void main(String[] args) throws Exception {
         ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[] {"META-INF/spring/dubbo-demo-consumer.xml"});
          context.start();
          DemoService demoService = (DemoService)context.getBean("demoService"); // 获取远程服务代理
          String hello = demoService.sayHello("world"); // 执行远程方法
          System.out.println( hello ); // 显示调用结果
      }
  }
  ```



## Dubbo配置的覆盖顺序以及推荐配置

Dubbo中一个配置项可以在多个地方进行配置，如果在多个地方同时配置了一个配置项，那么覆盖顺序是这样的

![](D:\documentary\mdImgs\Dubbo\3.jpg)

- 方法级优先，接口级次之，全局配置再次之（范围小的优先）。
- 如果级别一样，则消费方优先，提供方次之。



### Dubbo官方建议的配置

#### 1. 在 Provider 端尽量多配置 Consumer 端属性 

- 作服务的提供方，比服务消费方更清楚服务的性能参数，如调用的超时时间、合理的重试次数等
- 在 Provider 端配置后，Consumer 端不配置则会使用 Provider 端的配置，即 Provider 端的配置可以作为 Consumer 的缺省值 。否则，Consumer 会使用 Consumer 端的全局设置，这对于 Provider 是不可控的，并且往往是不合理的，可以让Provider端的实现者一开始就思考 Provider 端的服务特点和服务质量等问题。
- 建议在 Provider 端配置的 Consumer 端属性有：
  1. `timeout`：方法调用的超时时间
  2. `retries`：失败重试次数，缺省是 2 
  3. `loadbalance`：负载均衡算法 ，缺省是随机 `random`。还可以配置轮询 `roundrobin`、最不活跃优先`leastactive` 和一致性哈希 `consistenthash` 等
  4. `actives`：消费者端的最大并发调用限制，即当 Consumer 对一个服务的并发调用到上限后，新调用会阻塞直到超时，在方法上配置 `dubbo:method` 则针对该方法进行并发限制，在接口上配置 `dubbo:service`，则针对该服务进行并发限制

#### 2. 在 Provider 端配置合理的 Provider 端属性

1. `threads`：服务线程池大小
2. `executes`：一个服务提供者并行执行请求上限，即当 Provider 对一个服务的并发调用达到上限后，新调用会阻塞，此时 Consumer 可能会超时。在方法上配置 `dubbo:method` 则针对该方法进行并发限制，在接口上配置 `dubbo:service`，则针对该服务进行并发限制

#### 3.不要使用 dubbo.properties 文件配置，推荐使用对应 XML 配置[ ](https://dubbo.apache.org/zh/docs/v2.7/user/recommend/#不要使用-dubboproperties-文件配置推荐使用对应-xml-配置)

Dubbo 中所有的配置项都可以配置在 Spring 配置文件中，并且可以针对单个服务配置

#### 4. 服务接口尽可能大粒度

每个服务方法应代表一个功能，而不是某功能的一个步骤，否则将面临分布式事务问题，Dubbo 暂未提供分布式事务支持。

#### 5. 每个接口都定义版本号

每个接口都应定义版本号，为后续不兼容升级提供可能，如：

 `<dubbo:service interface="com.xxx.XxxService" version="1.0" />`。

建议使用两位版本号，因为第三位版本号通常表示兼容升级，只有不兼容时才需要变更服务版本。

当不兼容时，先升级一半提供者为新版本，再将消费者全部升为新版本，然后将剩下的一半提供者升为新版本



## Dubbo调用过程简要说明

### 框架设计

![](d:\documentary\mdImgs\Dubbo\4.jpg)

- config 配置层：对外配置接口，以 ServiceConfig, ReferenceConfig 为中心，可以直接初始化配置类，也可以通过 spring 解析配置生成配置类
- proxy 服务代理层：服务接口透明代理，生成服务的客户端 Stub 和服务器端 Skeleton
- registry 注册中心层：封装服务地址的注册与发现，以服务 URL 为中心
- cluster 路由层：封装多个提供者的路由及负载均衡，并桥接注册中心，以 Invoker 为中心
- monitor 监控层：RPC 调用次数和调用时间监控，以 Statistics 为中心
- protocol 远程调用层：封装 RPC 调用，以 Invocation, Result 为中心
- exchange 信息交换层：封装请求响应模式，同步转异步，以 Request, Response 为中心
- transport 网络传输层：抽象 mina 和 netty 为统一接口，以 Message 为中心
- serialize 数据序列化层：提供一些可复用的一些工具

### 服务暴露过程

![](d:\documentary\mdImgs\Dubbo\5.jpg)

### 服务引用过程

![](d:\documentary\mdImgs\Dubbo\6.jpg)

### 服务调用过程

![](d:\documentary\mdImgs\Dubbo\7.jpg)