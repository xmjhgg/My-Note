# Flink的定时器功能

## 定时器简介
Flink内置了定时器功能，提供在某个时间戳自动执行一段代码的能力  

在Flink的KeyedProcessFunction类中有一个可以重写的方法：onTimer()，这个方法便是flink提供的定时器功能，在这个方法内我们可以写想要在某个指定时间执行的代码，当flink的时间到达这个时间点时就会自动执行我们写在这个方法内的代码。  

## 如何使用定时器
**使用定时器的前提**：Flink定时器作用于KeyedProcessFunction的key，必须要经过keyBy的算子才可以使用flink的定时器，
在这个类的processElement()方法内，我们可以通过Context来获取到flink的timerService来注册一个定时器。  

- 注册方式：
  1. 调用context.timerService().registerProcessingTimeTimer(timestamp)。该方法注册的定时器执行时间是flink处理时间
  2. 调用context.timerService().registerEventTimeTimer(timestamp)。该方法注册的定时器执行时间是flink事件时间  

当时间到达定时器指定的时间时，就会调用onTimer()方法。  

（ProcessFunction中也有OnTimer方法，context中也可以获取到timerService()，但是不能注册定时器，这个方法的实现是直接抛出不支持的异常。并且在keyedSteam的process方法中ProcessFunction已经被标识为Deprecated。不能用于注册定时器）


## 定时器与keyBy中的key的关联
上面有说到定时器必须作用于keyBy之后的算子，那么定时器和key的

1. 每个key都可以注册定时器，不同key之间注册的定时器互不影响

2. 定时器执行onTimer()方法中，有三个入参：  
  - timestamp：定时器的执行时间  
  - TimerContext：定时器的上下文，主要看此参数，定时器的上下文中有一个方法是getCurrentKey()，此方法返回的就是注册这个定时器的key。  
  - Collector：记录收集器，将数据传递给下一个算子。

## 使用定时器时需要注意的地方
1. 每个key的每个时间戳只能有一个的定时器  
Flink内部维护了一个定时器队列，每个注册的定时器会添加到此队列中，添加时会按照key和时间戳去重，执行完会将此定时器移出队列。同一个key注册的定时器是串行执行：如果上一个定时器还在执行时，下一个定时器的触发时间已经到了，会等上一个定时器执行完再开始执行下一个
2. 定时器维护在堆内存中，使用定时器时需要注意这一方面的内存消耗
3. 如果注册了一个执行时间在过去的定时器，会立即执行此定时器
4. 可以在onTimer()方法中再注册定时器，达到固定时间间隔执行代码的效果
5. 定时器抛出异常，会导致flink停止


## 定时器使用样例
``` java 
public class TimeProcessFunction extends KeyedProcessFunction<String, AccountTrade, AccountTrade> {

    @Override
    public void processElement(AccountTrade value, KeyedProcessFunction<String, AccountTrade, AccountTrade>.Context ctx, Collector<AccountTrade> out) throws Exception {
        long currentTimeMillis = System.currentTimeMillis();
        long triggerTime = currentTimeMillis + 500;

        System.out.println(ctx.getCurrentKey() + "注册了：" + (triggerTime));
        ctx.timerService().registerProcessingTimeTimer(triggerTime);
        out.collect(value);
    }

    @Override
    public void onTimer(long timestamp, KeyedProcessFunction<String, AccountTrade, AccountTrade>.OnTimerContext ctx, Collector<AccountTrade> out) throws Exception {
        System.out.println(ctx.getCurrentKey() + "触发了：" + timestamp);
        Thread.sleep(100);
        System.out.println(ctx.getCurrentKey() + "执行完了：" + timestamp + " 。当前时间：" + System.currentTimeMillis());
    }

}
```


### 待解决疑问
1. 为什么flink要把定时器和key绑定在一起，为何定时器不能无视key单独运行？  
如果就是要固定执行某个和key无关的定时器，比如每30分钟固定输出当前系统时间，那么这时候的的做法就只能是在keyBy时，根据一个常量字符串keyBy，固定只有一个key注册定时器。定时器应该不必要和key强耦合在一起，不太理解。  
也许是因为在流式处理中，定时器大多数时候的执行代码都是和key相关联的，比如收到某个订单的事件，定时刷新这个订单的状态等等。

