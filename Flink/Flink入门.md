# Flink入门总结



## Flink是什么



> Apache Flink is a framework and distributed processing engine for stateful computations over *unbounded and bounded* data streams

Flink是一款计算有状态的有界数据流和无界数据流的分布式框架引擎

*状态*：将数据集成一个大的集合中，在内存中维护这个数据集合的正确性，这个数据集合就是状态

*数据流*：将数据输入抽象比喻成像流水一样，源源不断的输入到计算机中（用户不停的注册、下单）

*有界：*有界是指数据的输入可以划分界限，离线的数据就是有界的，比如用户观看某个课程，看到结束，那么这个数据就是有界的，也可以人为的划分数据的界限，比如以某一个月内的数据作为界限，

*无界*：数据的输入是没有界限的，实时的数据是无界的，数据源源不断的输入到计算机中，比较符合现实情况





## Flink简单上手开发

- 引入jar包

  ```java
  <dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-java</artifactId>
      <version>1.10.1</version>
  </dependency>
  
  <dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-streaming-java_2.12</artifactId>
      <version>1.10.1</version>
  </dependency>
  ```

- FlatMapFunctionAPI说明

  ```java
  //在这个接口中需要定义两个泛型，第一个泛型是数据的输入，第二个泛型是数据的输出
  //Tuple2 ：二元组，在这个案例中的存储结构为("a",1)
  class MyFlatMapFunction implements FlatMapFunction<String, Tuple2<String,Integer>>{
  	
      //FlatMapFunction的核心方法flatMap，这个方法的第一个参数是输入的数据,第二个参数是输出的数据，是一个集合
      public void flatMap(String value, Collector<Tuple2<String, Integer>> out) throws Exception {
              String[] words = value.split(" ");
              for (String word : words) {
                  //调用cllect方法将这次的处理结果加入到输出集合中
                  out.collect(new Tuple2<String, Integer>(word,1));
              }
          }
      }
  ```

- 数据批次式处理开发，批次式将数据累积到一定程度后才开始处理

  ```java
  public static void main(String[] args) throws Exception {
  	//批次式处理的Flink运行环境
      ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
  	
      String inputPath = "E:\\source.txt";
  	
      //获取数据集合，DataSource继承了DateSet
      DataSource<String> inputSource = env.readTextFile(inputPath);
  
      inputSource.flatMap(new MyFlatMapFunction())
                  .groupBy(0)//以输出集合的第一个位置进行分组
                  .sum(1)//累加每个输出集合的第二个数
                  .print();//打印
  
      }
  ```

- 数据流式处理开发，流式是数据一输入就立即处理

  ```java
  public static void main(String[] args) throws Exception {
  
  StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
  
      String inputPath = "E:\\source.txt";
  	
      //获取数据流，DataStreamSource继承了DataStream
      DataStreamSource<String> inputSource = env.readTextFile(inputPath);
      
      inputSource.flatMap(new MyFlatMapFunction())
      //和批次式处理不同的是 流式处理是keyBy 因为流式中没有一次处理多个数据，所以也就没有将数据分组的概念
          .keyBy(0)
          .sum(1)
          .print();
      
  	//流式处理需要启动执行流式的执行环境
      env.execute();
      }
  ```

  

## Flink部署



### 单机版部署

1. Flink需要先配置好jdk
2. 下载Flink wget mirrors.bfsu.edu.cn/apache/flink/flink-1.11.3/flink-1.11.3-bin-scala_2.12.tgz)
3. 解压 tar -zxvf flink1.1.3
4. 启动 ./start-cluster.sh

2. 访问8081端口即可查看当前的Flink运行情况



### Flink配置文件 - flink-conf.yaml

- jobmanager.rpc.address：jobmanger远程通信时的地址
- jobmanager.rpc.address：jobmanger远程通信的端口
- jobmanager.memory.process.size：jobmanager可持有的内存(包括JVM的内存和直接内存)，jobmanager负责任务的调度
- taskmanager.memory.process.size:taskmanger可持有的内存(包括JVM的内存和直接内存)，taskmanger真正执行程序，所以这个配置值应该比jobmanger的内存大
- taskmanager.numberOfTaskSlots:每个taskManager 提供的 slots 数量大小（每个task最大并行度的大小）
- parallelism.default：taskManager 默认的并行度



## Flink角色

### 任务提交流程

一个Flink任务的提交流程如下

![](E:\documentary\mdImgs\Flink\Flink入门\1.jpg)

### 角色

- **JobManager**：作业管理者

  - JobManager是控制（控制，不是执行）程序执行的主进程，每个提交的Job都会被不同的JobManager管理

  - JobManager会先收到要执行的应用程序，这个应用程序会包括作业图（JobGraph），逻辑数据流图（logical dataflow graph）和打包的所有的类库和其它资源的JAR包

  - JobManager会根据程序的作业图生成一个真正可执行的执行图

    JobManager会向ResourceManager申请执行任务需要的资源，也就TaskManager的资源管理器的Slot（插槽），有了足够的资源后JobManager就会将执行图发到真正运行它们的TaskManager上

- **TaskManager**：任务管理者

  - Flink中通常会有多个TaskManager运行，每个TaskManger有一定数量的Slot，插槽的数量决定TaskManger能执行的任务数量
  - TaskManager启动后会向Resource Manager注册自己有多少个Slot资源；在收到资源管理器的指令后TaskManager就会将slot提供给JobManager调用，JobManger就会将任务分配给slot来执行任务
  - TaskManager可以跟其他运行同一个任务的TaskManager交换数据

- **Resource Manager**：资源管理者

  - 资源管理器的主要主要作用是管理TaskManager的slot
  - Flink为不同的环境提供了不同的资源管理器，比如YARN，K8s，单机部署等
  - 当JobManager向ResourceManager申请资源时，如果ResourceManager没有足够的插槽来满足JobMnager的需求，ResourceManager可以向资源提供平台发起会话来启动一个新的TaskManager进程的容器

- **Dispacher**：分发器

  - 主要作用负责任务的提交，会将任务移交给一个JobManager
  - Dispacher会启动一个Web UI来展示任务的执行情况
  - Dispacher不是必须的，取决于任务的提交方

## Window API的使用

###  windwo简介

一般的数据流都是无界的，而窗口就是将无界的数据流进行切割，得到有界的数据流，而窗口就是Flink提供的切割工具，窗口（window）会将流数据分发到有限大小的桶（bucket）中，然后再对桶内的数据进行批量处理

将流数据分发到有限大小的桶中，仍然是流式处理而不是批处理

### window类型

Flink中的window有两类

- 时间窗口

时间窗口又可以分为3类

- 滚动时间窗口

滚动时间窗口将数据用固定大小的窗口，以窗口大小的长度移动来对数据进行切分，切分的数据没有重叠

- 滑动时间窗口

滑动时间窗口是固定大小的窗口，自定义的窗口移动长度，切分的数据可能会出现重叠（移动的长度小于窗口的大小），滚动时间窗口也可以理解成一种滑动时间窗口

- 会话窗口

会话窗口是设置一个时间间隔，当会话窗口超过这个间隔还没有收到数据的话，当前的窗口就会将数据分发到桶中，然后当前的窗口就关闭。当新的数据到来后会再生成一个新的窗口

- 计数窗口

计数窗口和时间窗口类似，只是技术窗口是以数据流的个数来进行划分，而不是时间划分

计数窗口只有两种

- 滚动计数窗口
- 滑动计数窗口



### 代码使用

以滚动时间窗口为例

```java
StreamExecutionEnvironment env= StreamExecutionEnvironment.getExecutionEnvironment();

DataStreamSource<String> dataStreamSource = env.socketTextStream("localhost", 7777);

dataStreamSource.flatMap(new SensorFlaMapFunction())
.keyBy("name")
.timeWindow(Time.seconds(15))
.aggregate(new SensorCountAggregateFunction())
.print();

env.execute();


//aggregate方法类 createAcccum
public static class SensorCountAggregateFunction implements AggregateFunction<Sensor, Integer, Integer>{

public Integer createAccumulator() {
return 0;
}

public Integer add(Sensor value, Integer accumulator) {
return accumulator + 1;
}

public Integer getResult(Integer accumulator) {
return accumulator;
}

public Integer merge(Integer a, Integer b) {
return a + b;
}
}
```

- 注意：每次窗口会将数据放入桶中，再将桶中的数据一起处理，如果时间窗口比较小，每次调用聚合方法计算传入的文本个数是这个窗口时间放入桶中的文本个数(之前窗口大小只有1秒，每次累加函数的结果都是1)




## 时间语义与WaterMark

### 时间语义

Flink中有三种时间语义：

- Event Time：事件创建时间，这个时间比较关键，Flink任务中一般多采用这个做为时间
- Ingestion Time：数据进入Flink的时间
- Processing Time：执行操作算子的本地机器时间



### WaterMark

引入了窗口和时间语义的概念那么我们就需要对下面这种情况进行处理

如果我们采用EventTime模式来处理事件，那么每个事件的数据可能会因为网络传输而到达的顺序和事件发生的顺序不同（比如：订单创建事件比订单支付事件晚到达Flink），这时我们就需要判断时间窗口要在事件语义的模式下什么时候去关闭这个窗口，将桶内的数据分发出去比较合适



*eg*:开了一个5秒的时间窗口，而接收到的事件内的事件时间为 1 2 5，这时候当窗口接收到事件时间为5的事件时，就关闭窗口了，但是 还有3 4两个事件时间的事件没有接收到，乱序会导致窗口计算时间不准确



Flink种提供了 **WaterMark **来对迟到的事件数据来处理

- WaterMark对窗口计算时间乱序的处理方案

  当窗口接收到要关闭窗口的事件时，不立即关闭窗口，而是等待一定时间，等迟到的数据来了再关闭，WaterMark的作用就是衡量这个等待时间来决定关闭什么窗口

  ​	*eg*: 我们设置一个WaterMark 为3秒 当接收到事件时间为5秒的事件时 ，我们认为 5-3 2秒之前的事	件都已经到达了，那么0-2秒的时间窗口就可以关闭了

  - 如何选择合适的WaterMark

    WaterMark的主要作用时处理短时间延迟的事件。WaterMark如果设置的过大，会导致系统整体处理事件变慢，所以需要选择一个合适的WaterMark。

    选择方法一般时通过观察系统内大多数事件的延迟情况，选择延迟时间最大的两个事件间隔

    ​	*eg*：比如事件时间顺序为7 5 1，那么WaterMark应当设置为4

- WaterMark的执行过程

  - WaterMark是一条含有时间戳的特殊的数据记录，可以像事件一样被Flink接收处理

  - WaterMark必须单调递增，保证事件时间在不停的推进

  - 执行过程

    1. WaterMark初始时为负数，表示没有任何事件接收处理

    2. 当一个事件被Flink接收处理时那么就会去读取这个事件的事件时间，

       如果这个事件时间-WaterMark设置的值大于当前WaterMark的时间戳，那么就会更新WaterMark的时间戳

       - 举例

         有一组事件的事件时间（左进右出）

         一个左进右出的队列中，有以下几个事件，他们的事件时间分别为

         8 3 2  4 1

         然后我们有一个长度为秒的时间窗口

         WaterMark设置为3

         那么WaterMark将会以如下方式执行

         - 初始，还没事件出队列

           WaterMark处于初始状态 值为-3

         - 事件时间1进入

           WaterMark检测事件时间是否大于当前的WaterMark， 1- 3 > 0-3 所以

           WaterMark将值更新为 1-3 = -2

         - 事件时间4进入

           WaterMark将值更新为 4-3 = 2

         - 事件时间5进入

           WaterMark将值更新为5-3 = 2

         - 事件时间2进入

           由于2-3<当前WaterMark的时间戳，所以不会更新WaterMark的值

         - 事件时间3进入

           同上，小于档期那WaterMark的值不做处理

         - 事件时间8进入

           WaterMark将记录的时间戳更新为5

         - WaterMark进入

           WaterMark时间戳为5，此时才会关闭开启的长度为5的时间窗口

         ### WaterMark在并行的下游子任务中的传递

         Flink数据流会在有可能传输给并行的下游子任务（比如执行keyBy后的操作），所以我们还需要注意当下游子任务接受到WaterMark时会如何处理

         与上面一样，每个并行的下游子任务都有一个初始的WaterMark的值，当从上游接受到WaterMark的值时，下游的子任务将会选择更新值最小的WaterMark（因为可以保证这个最小的WaterMark之前的数据已经到达），并将这个被更新之前的WaterMark发送给更下游的子任务

         

         ### 在代码中使用WaterMark

         ```java
         	/**
             * @Descrept :WaterMark测试
             */
             @Test
             public void Test2 () throws Exception {
         
                 StreamExecutionEnvironment env= StreamExecutionEnvironment.getExecutionEnvironment();
         
                 DataStreamSource<String> dataStreamSource = env.socketTextStream("47.102.222.110", 7777);
         
                 SingleOutputStreamOperator<Sensor> dataStream2 = dataStreamSource.flatMap(new SensorFlaMapFunction())
                         //在构造方法中指定WaterMark要延迟的时间
                         .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<Sensor>(Time.seconds(2)) {
         
                             //指定元素中哪个属性为事件时间
                             @Override
                             public long extractTimestamp(Sensor sensor) {
                                 return sensor.getTime();
                             }
                      });
         
                 //统计事件时间15秒内的最小温度
                 dataStream2.keyBy("name")
                         .timeWindow(Time.seconds(15))
                         .minBy("temperature")
                         .print();
         
         
                 env.execute();
             }
         ```

         - 窗口的起始位置

           在开窗口时，第一个接收到事件时间不一定就时从0开始，这时候就需要确认第一个窗口的左边界在哪

           Flink中计算起始位置的代码如下

           ```java
           public static long getWindowStartWithOffset(long timestamp, long offset, long windowSize) 
           {
           		return timestamp - (timestamp - offset + windowSize) % windowSize;
           }
           ```

           也就是时间戳对窗口大小取模




## Flink中的状态管理



### 状态的定义

Flink中任务的执行有时侯时需要依赖之前的任务执行的结果，这是我们就需要维护一个数据结构，这个数据结构就被称为状态

- 状态的定义：
  - 由一个任务维护，并且用来计算某个结果的所有数据，都属于这个任务的状态
  - 可以认为状态就是一个本地变量，可以被任务的业务逻辑访问
- Flink 会自己进行状态管理，包括状态一致性、故障处理以及高效存储和访问，以便开发人员可以专注于应用程序的逻辑，开发人员只需要专注于应用程序的逻辑（我们无需关注任务并行度提高或者减少时，状态没有增加或者时合并，以及发生故障时状态的恢复）



### 状态的类型

Flink中有两种状态类型

- 算子状态

  - 算子状态的作用范围限定在算子内，由算子进行维护，同一并行的任务都可以访问到相同的状态

  - 算子状态对同一任务的子任务是共享的
  - 算子状态不能不同任务的的子任务访问

  

- 键控状态

  - 键控状态根据数据流中指定的key来维护
  - Flink为每个key维护一个状态实例，将具有相同key的数据都分区到同一个算子任务中，这个任务会维护和处理这个key对应的状态
  - 当有任务要处理一条数据时，这个任务会自动将状态的访问权限设定为当前数据的key



### 算子状态的代码使用

```java
 	//这里在FaltMap中维护了一个接收到了多少个事件的状态，状态的维护的关键就是实现ListCheckpointed接口
    public static class SensorCountFlatMapFunction implements  FlatMapFunction<String , Integer>, ListCheckpointed<Integer>{

        private Integer count = 0;

        @Override
        public void flatMap(String value, Collector<Integer> out) throws Exception {
            count ++;
        }

        //状态的快照 - 用于备份状态
        @Override
        public List<Integer> snapshotState(long checkpointId, long timestamp) throws Exception {
            return Collections.singletonList(count);
        }

        //状态的回复 - 当发生故障时执行的回复状态的方法， 参数就是前面的方法备份的数据
        @Override
        public void restoreState(List<Integer> state) throws Exception {
            for (Integer num : state) {
                count += num;
            }
        }
    }
```



### 键控状态的代码使用

```java
public static class SensorKeyStateMapFunction extends RichMapFunction<Sensor,Integer>{

        ValueState<Integer> count;

        @Override
        public Integer map(Sensor value) throws Exception {
            count.update(count.value() +1);
            return count.value();
        }

        @Override
        public void open(Configuration parameters) throws Exception {
            //键控状态的关键在于要获取当前任务的执行环境，将不同的key的状态分开维护，所以这里需要通过RichFunction获取执行环境的上下文
             count = getRuntimeContext().getState(new ValueStateDescriptor<Integer>("myKeyState",Integer.class));
             count.update(0);
        }
    }
```



### 状态后端的组件选择

对于有状态的算子，每次算子在进行任务时，都会读取和更新状态，所以选择一个适合的维护状态的组件也非常重要，这个维护状态的组件称为状态后端

状态后端主要实现的功能就是两点

- 本地状态的管理
- 将检查点的状态持久化

Flink提供了三种维护状态的组件

- MemoryStateBackend

  - 内存级的状态后端，会会将键控状态作为内存中的对象进行管理，将它们存储在

    TaskManager 的 JVM 堆上，而将 checkpoint 存储在 JobManager 的内存

    中 ，将本地状态和检查点都维护在内存中

  - 特点：快速、低延迟，但不稳定

- FsStateBackend

  - 将 checkpoint 存到远程的持久化文件系统（FileSystem）上，而对于本地状

    态，跟 MemoryStateBackend 一样，也会存在 TaskManager 的 JVM 堆上

    本地状态在内存中，检查点持久化到远程的系统中

  - 特点：同时拥有内存级的本地访问速度，和更好的容错保证，是比较好的选择

- RocksDBStateBackend

  - 将本地状态和检查点持久化到本机的一个NOSQL存储中
  - 特点：可以支持非常大量的本地状态（不会因为内存限制本地状态的大小，状态存储于硬盘中，内存中有缓存一部分状态），但是由于需要经常持久化的原因访问速度较慢

状态的代码配置：

```java
<dependency>
 <groupId>org.apache.flink</groupId>
 <artifactId>flink-statebackend-rocksdb_2.12</artifactId>
 <version>1.10.1</version>
</dependency>
```



```java
StreamExecutionEnvironment env = 
StreamExecutionEnvironment.getExecutionEnvironment();
env.setParallelism(1);
// 1. 状态后端配置
env.setStateBackend(new FsStateBackend(""));
// 2. 检查点配置
env.enableCheckpointing(1000);
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
env.getCheckpointConfig().setCheckpointTimeout(60000);
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(500);
env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);
env.getCheckpointConfig().setPreferCheckpointForRecovery(false);
env.getCheckpointConfig().setTolerableCheckpointFailureNumber(0);
// 3. 重启策略配置
// 固定延迟重启（隔一段时间尝试重启一次）
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
 3, // 尝试重启次数
 100000 // 尝试重启的时间间隔
))
```





## ProcessFunction API的使用

ProcessFunction是Flink提供的更底层的API，在开发使用中可以获取到更多的Flink运行时的信息，处理数据也更为灵活，还提供定时器的功能

Flink提供了8个Process Function：

- ProcessFunction
- KeyedProcessFunction（Key Stream）
- CoProcessFunction(Conect Stream，两条数据Conect后使用的ProcessFunciton)
- ProcessJoinFunction（两条数据流join后使用的ProcessFunction）
- BroadcastProcessFunction(广播流)
- KeyedBroadcastProcessFunction(keyBy后的广播流)
- ProcessWindowFunction（窗口使用的ProcessFunction）
- ProcessAllWindowFunction(没有KeyBy的窗口使用的ProcessFunction)

使用什么样的ProcessFunction取决于数据流是什么形式

### 基础代码开发的使用

以ProcessFunction为例

```java
public abstract class ProcessFunction<I, O> extends AbstractRichFunction{
    //...
}
//ProcessFunction继承了RichFuntion,所以我们可以获取到运行时的上下文Context，通过context 我们可以获取到当前的处理时间、当前的分区key、当前的watermark等等...

public void onTimer(long timestamp, OnTimerContext ctx, Collector<O> out) throws Exception{}
//ProcessFuntion还实现了定时器的方法，我们可以在这里定义自己的定时器需要执行的逻辑


```

ProcessFunction最基础的使用

```java
dataStreamSource.process(new ProcessFunction<Sensor, Integer>() {
            int i = 0;

            @Override
            public void processElement(Sensor value, Context ctx, Collector<Integer> out) throws Exception {

                if (value.getTemperature() > 36.5){
                    ctx.output(outputTag,value);
                }else {
                    out.collect(i + 1);
                }

            }
        }
```



## Flink容错机制

### 检查点（check point）

Flink容错机制的核心就在于检查点

- 检查点应该在什么时候去存储状态？

  这个时间点应当每个子任务都处理完同一份数据的时候，这样不论从哪个事件开始重新恢复Flink都可以将数据还原回来

### 检查点的实现算法



Flink中每个算子的执行的子任务状态都不同，如果来进行统一的状态存储？

Flink采用了分区式的异步快照存储，每个子任务在执行完当前的任务后就会去进行一次单独检查点的快照存储，然后当每个子任务所属当前的任务都执行完了，Flink会将这些单独检查点存储的快照拼接在一起。

这样做的好处就是每个子任务可以不间断的继续执行当前的处理任务，而不需要因为要进行快照存储等待还没有执行完的子任务执行完成

那么要如何判断子任务是否执行完了当前的任务？Flink提供了一个类似于WaterMark的数据，叫做检查点分界线，当任务接收到检查点分界线时，就说明当前任务已经执行完成，就去进行一次快照存储

- 检查点分界线（Check Barrier）

  分界线之前到的数据导致的状态更改，都会包含在当前分级先所属的检查点中；而基于分界线之后的数据导致的所有更改，就会被包含在之后的检查点中



- 检查点分界线在下游子任务的传递

  当一个任务处理完Barrier后会，会将这个barrier广播给所有的下游子任务

  每个下游子任务需要等收到它的所有上游任务的Barrier才会去进行快照存储（这是因为分界线之前的数据可能经过keyBy或者是分流，转交给不同的上游任务处理，等收到所有的上游任务的Barrier才能保证这个Barrire之前的所有事件都被上游处理好了）

  - 这时又会有一个问题，如果同一个任务的下游子任务在收到Barrier后，又立刻收到了来自上游处理好的数据时，会怎么做？

    这时候的下游子任务会先将来自上游处理好的数据缓存起来，等收集完Barrier，进行快照存储后才继续处理来自上游的数据，这样才能分界线的有效，保证每次存储状态的一致性。

### 保存点(Save point)

Flink除了自带的检查点可以保存状态，还提供了自定义的功能去保存状态

这个自定义的功能就是保存点

- 保存点

  保存点使用的算法和检查点完全相同，区别只在于Flink不会自动的创建保存点，需要开发人员明确的在某个地方去创建保存点的操作

- 保存点的用途

  保存点在除了用于故障恢复外，还可以用于有进化的手动备份、更新Fkink程序，版本迁移，暂停程序，重启等等操作，是一个强大的功能

  

###  恢复状态

当程序异常，系统需要重新恢复状态时，Flink会从领近检查点开始恢复状态，将检查点的状态恢复

### sink端的容错机制

- 幂等写入

  保证每次代码被重复执行时得到的结果一致

- 事务写入

  当checkPonit保存完成时，才真正执行sink操作

  - 预写日志

    在执行事务之前预先写入到日志中，当收到了checkPoint完成的通知，将日志中的所有数据一起写入

  - 两阶段提交

    真正启动事务，当收到checkpoint事件了，sink启动一个事务，将接下来收到到的数据添加到事务中，当收到了checkpoint完成的通知，才真正的写入数据

    保证每次checkponit的数据都正确的持久化了

- Flink Exactly-once

  Exactly-once 就是保证每个事件只会消费一次

  Exactly-once 的执行需要三端（事件数据的来源端，Flink处理端（Flink已经支持），Sink端）的配合

  比较常见的配置就是

  Kafak - flink - Kafak

  

## Table API的使用

Table ： 将数据流抽象成如同mysql中的表一样使用

### TableAPI是什么

Flink本身是批流统一处理的框架，TableAPI是Flink提供的统一的对流处理和批处理的的上层封装，将数据流像表一样的使用，可以非常的直观的对数据流来进行一些表的关系运算符的查询

TableAPI支持SQL查询，也可以使用Flink提供的查询API对数据流转换成的表进行查询



### TableAPI和Flink SQL代码使用

首先需要引入相关jar包

```java
<!--计划器，Table API 最主要的部分，提供了运行时环境和生成程序执行计划的-->
<dependency>
   <groupId>org.apache.flink</groupId>
   <artifactId>flink-table-planner_2.12</artifactId>
   <version>1.10.1</version>
</dependency>

<!--bridge 桥接器，主要负责 Table API 和 DataStream/DataSet API的连接支持-->
<dependency>
   <groupId>org.apache.flink</groupId>
   <artifactId>flink-table-api-java-bridge_2.12</artifactId>
   <version>1.10.1</version>
</dependency>
```



- 快速开始

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();

DataStreamSource<Sensor> sourceStream = env.addSource(new MySource());

StreamTableEnvironment tableEnvironment = StreamTableEnvironment.create(env);

Table dataTable = tableEnvironment.fromDataStream(sourceStream);

//TableAPI 执行后的Table
Table apiTable = dataTable.select("id,temperature")
        .where("id = 'sensor_1'");

tableEnvironment.createTemporaryView("sensor",apiTable);

//SQL执行后的Table
Table sqlTable = tableEnvironment.sqlQuery("select id,temperature from sensor where id ='sensor_1'");

tableEnvironment.toAppendStream(apiTable,Row.class).print("api");
tableEnvironment.toAppendStream(sqlTable, Row.class).print("sql");

env.execute();
```



#### Table的基本程序结构

Table API 和 SQL 的程序结构，与流式处理的程序结构类似；也可以近似地认为有这么几步：首先创建执行环境，然后定义 source、transform 和 sink。

```java
// 创建表的执行环境
StreamTableEnvironment tableEnv = ... 
// 创建一张表，用于读取数据
tableEnv.connect(...).createTemporaryTable("inputTable");
// 再创建一张表，用于把计算结果输出，这三步可以视为source
tableEnv.connect(...).createTemporaryTable("outputTable");
// 通过 Table API 查询算子，得到一张结果表
Table result = tableEnv.from("inputTable").select(...);
// 通过 SQL查询语句，得到一张结果表，这两步可视为transform
Table sqlResult = tableEnv.sqlQuery("SELECT ... FROM inputTable ...");
// 将结果表写入输出表中，视为sink
result.insertInto("outputTable");
```



#### 详细使用

##### 表环境

表环境（TableEnvironment）是 flink 中集成 Table API & SQL 的核心概念。

它负责:

- 注册 catalog

- 在内部 catalog 中注册表

- 执行 SQL 查询
- 注册用户自定义函数
- 将 DataStream 或 DataSet 转换为表
- 保存对 ExecutionEnvironment 或 StreamExecutionEnvironment 的引用



其中表环境中使用的planner（桥接器）有新旧之分，以1.13.0版本为界

新旧的主要区别：

新的planner称为Blink，Blink 将批处理作业，视为流式处理的特殊情况。所以，blink 不支持表和DataSet 之间的转换，批处理作业将不转换为 DataSet 应用程序，而是跟流处理一样，转换为 DataStream 程序来处理。

- 代码配置表环境采用blink或者是oldPlanner

  ```java
  //使用blink
  EnvironmentSettings bsSettings = EnvironmentSettings.newInstance()
  .useBlinkPlanner()
  .inStreamingMode().build();
  StreamTableEnvironment bsTableEnv = StreamTableEnvironment.create(env, 
  bsSettings);
  ```

  ```java
  //使用老版本planne
  EnvironmentSettings settings = EnvironmentSettings.newInstance()
   .useOldPlanner()
   .inStreamingMode()
   .build();
  StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env, 
  settings);
  ```

#####  Catalog - 表

与MYSQL不同的是 Flink的Table多了一层Catelog的概念，表环境可以注册目录 Catalog，并可以基于 Catalog 注册表。它会维护一个Catalog-Table 表之间的 map。 

表（Table）是由一个“标识符”来指定的，由 3 部分组成：Catalog 名、数据库（database）名和对象名（表名）。

如果没有指定目录或数据库，就使用当前的默认值。表可以是常规的（Table，表），或者虚拟的（View，视图）。常规表（Table）一般可以用来描述外部数据，比如文件、数据库表或消息队列的数据，也可以直接从 DataStream 转换而来。视图可以从现有的表中创建，通常是 table API 或者 SQL 查询的一个结果。

##### Table连接第三方数据

连接外部系统在 Catalog 中注册表，直接调用 tableEnv.connect()就可以，里面参数要传入一个 ConnectorDescriptor，也就是 connector 描述器

- 连接文件系统

  对于文件系统的 ConnectorDescriptor而言，flink内部已经提供了，就叫做 FileSystem()。

  ```java
  <dependency>
   <groupId>org.apache.flink</groupId>
   <artifactId>flink-csv</artifactId>
   <version>1.10.1</version>
  </dependency>
  ```

  ```java
  tableEnv
   .connect( new FileSystem().path("sensor.txt")) // 定义表数据来源，外部连接
   .withFormat(new Csv()) // 定义从外部系统读取数据之后的格式化方法
   .withSchema( new Schema()
   .field("id", DataTypes.STRING())
   .field("timestamp", DataTypes.BIGINT())
   .field("temperature", DataTypes.DOUBLE())
    ) // 定义表结构
   .createTemporaryTable("inputTable"); // 创建临时表
  ```

- 连接kafka

  ```java
  tableEnv.connect(
   new Kafka()
   .version("0.11") // 定义 kafka 的版本
   .topic("sensor") // 定义主题
   .property("zookeeper.connect", "localhost:2181")
   .property("bootstrap.servers", "localhost:9092") )
   .withFormat(new Csv())
   .withSchema(new Schema()
   .field("id", DataTypes.STRING())
   .field("timestamp", DataTypes.BIGINT())
   .field("temperature", DataTypes.DOUBLE())
  )
   .createTemporaryTable("kafkaInputTable");
  ```

##### 表的查询

- TABLE API查询

  普通查询

  ```java
  Table sensorTable = tableEnv.from("inputTable");
  Table resultTable = senorTable
  .select("id, temperature")
  .filter("id ='sensor_1'");
  ```

  聚合查询

  ```java
  Table aggResultTable = sensorTable
  .groupBy("id")
  .select("id, id.count as count");
  ```

- SQL查询

  普通查询

  ```java
  Table resultSqlTable = tableEnv.sqlQuery("select id, temperature from 
  inputTable where id ='sensor_1'");
  ```

  聚合查询

  ```java
  Table aggResultSqlTable = tableEnv.sqlQuery("select id, count(id) as cnt from
  inputTable group by id");
  ```

##### DataStream转换成表

Table 和 DataStream 做转换：

我们可以基于一个 DataStream，先流式地读取数据源，然后 map 成 POJO，再把它转成 Table。Table 的列字段（column fields），就是 POJO 里的字段。

- POJO映射代码使用

  ```java
  Table sensorTable = tableEnv.fromDataStream(dataStream, "id, timestamp.rowtime as ts, temperature");
  ```

  也可以通过 as 对字段重命名

- 也可以通过scheme映射，例子参考Table连接第三方数据

##### 创建视图

```java
//直接将POJO转换为视图
tableEnv.createTemporaryView("sensorView", dataStream);

//将POJO转换为视图，并指定需要的字段
tableEnv.createTemporaryView("sensorView", dataStream, "id, temperature, timestamp as ts");
                             
//将表转换为视图
tableEnv.createTemporaryView("sensorView", sensorTable);
```

##### 输出表

输出表示将表中的数据进行输出，对应流数据处理的sink阶段

```java
/ 注册输出表
tableEnv.connect(
 new FileSystem().path("…\\resources\\out.txt")
) // 定义到文件系统的连接
 .withFormat(new Csv()) // 定义格式化方法，Csv 格式
 .withSchema(new Schema()
 .field("id", DataTypes.STRING())
 .field("temp", DataTypes.DOUBLE())
) // 定义表结构
 .createTemporaryTable("outputTable"); // 创建临时表
resultSqlTable.insertInto("outputTable");
```





### Table的更新模式







### Table连接第三方数据库







### Table和DataStream转换

#### 动态表

因为Flink中的数据都是无界的 ，所以在对Flink生成的表进行查询时，会生成一张持续变化的结果表



### Table与窗口组合

#### Table的时间特性

除了Flink指定的时间特性，Flink生成的Table也可以指定时间特性

- 处理时间（Process Time）

  可以在代码的schema中使用 .proctime即可

- 事件时间（Even Time）

  rt.rowtime

#### Group Windows

#### Over Windows

### Table用户自定义函数



#### 标量函数

标量函数是 来一行数据，返回一行数据

必须自己创建实现一个名字为 eavl 的方法

#### 表函数

表函数是 来一行数据，可以返回多行数据

必须自己创建实现一个名字为 eavl 的方法

#### 聚合函数

必须自己创建实现一个名为accumulate的方法

聚合函数对航航数据聚合返回单行数据

#### 表聚合函数

表聚合函数可以对多行数据进行聚合返回多行数据

AggregationFunciton必须自己创建实现的方法名

createAccumulator（创建初始的聚合状态，初始的空累加器）

accumulate（聚合计算过程，累加器）

emitValue（获取聚合状态）











