> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=MzkxNjA1MzM5OQ==&mid=2247495719&idx=1&sn=babd539653a0abcef8eb43429fde2785&chksm=c1577cdff620f5c948d9cd5db7ce6c3fb99767afab714ee2fd4cb99b2ec57a5d92f3a68f0be0&mpshare=1&scene=24&srcid=0328zfzsUjbi7zesFgW79PS7&sharer_sharetime=1648429892492&sharer_shareid=19c1c7c6f7a5855b169667fa4dc9b424&key=a4d39c7b611bb8e1eea92d84da42402adca20953af158e0bf879e389a71fc16f68ca8bf2e04e9ae33e627479829d40d4b27528ebd99a7f933f09be8035504e2da62f8aff51503b5c91a1a2d42d4559b3c872d77cece65566b20498ece4c2fcc9e0c91282c71b6ca7f8ca0e3f15d253493552459222bc789af37081a945bd33bf&ascene=14&uin=NzI1NTIwNTYy&devicetype=Windows+XP&version=62060841&lang=zh_CN&exportkey=AVHGy38QEQfHuA0CpxeEwyg%3D&acctmode=0&pass_ticket=caHajDsHePwDrUwfEiBfjj3IlkK2DNZulTwZoqFIVYf8qaJBr9P7UXLqOHGCSBxL&wx_header=0)

1. 前言  

========

大家好，我是**老羊**，本文主要是整理博主收集的 Flink 高频面试题。之后**每周**都会有一篇。如果本文对你有所帮助，请点个**喜欢 + 在看**吧。

这一期的面试题主要是介绍 Flink 面试中的高频面试题，Flink 流 Join 相关内容，相信大家在面试中遇到的太多了，本节包含的主要内容如下：

1.  **⭐ Join 的应用场景**
    
2.  **⭐ 为什么流式计算中提到 Join 小伙伴萌就怕呢？**
    
3.  **⭐ 带大家看一遍本文思路**
    
4.  **⭐ Flink Join 解决方案：Flink Window Join**
    
5.  **⭐ Flink Join 解决方案：Flink Interval Join**
    
6.  **⭐ Flink Join 解决方案：Flink Regular Join**
    
7.  **⭐ 上述 3 种解决方案各有优劣，有没有什么共性的问题可以优化？**
    
8.  **⭐ Flink Join 优化方案：同 key 共享 State**
    
9.  **⭐ Flink Join 优化方案：外存 State 之 Redis**
    

下面的答案都是博主收集小伙伴萌的答案 + 博主自己的理解进行的一个总结。

2.Join 的应用场景
============

关于 Join 的场景就太多太多了，在离线数仓开发中，Join 是最常用的算子之一了。

比如：

1.  ⭐ 几乎所有公司的 APP 都会涉及到的曝光关联点击；两条流数据之间的维度拼接；将表打宽等等
    
2.  ⭐ 电商场景中的退单的订单关联下单的订单分析退单的单的特点等
    

3. 为什么流式计算中提到 Join 小伙伴萌就怕呢？
===========================

很多离线数仓的小伙伴会说，Join 这玩意**非常简单**啊，Hive SQL 简简单单的写个关联 SQL 就行啊。

是的，在批式计算中，Join 的左右表都是 "全集"，所以在全集上面做关联操作是非常简单的，比如目前离线中的技术方案有 sort-merge、hash join 等，这些方案都非常成熟了，哪怕博主自己写个 Java 代码也能实现一个极简版本的批 Join。

但是，在流式计算中，左右表的数据都是无界的，而且是实时到来的。这就会引起流式计算中的 2 个问题 + 大数据中的 2 个核心问题（我们以 A left join B 举例）：

流式计算中的 2 个问题：

1.  **_⭐ 流式数据到达计算引擎的时间不一定：_**比如 A 流的数据先到了，A 流不知道 B 流对应同 key 的数据什么时候到，没法关联（数据质量问题）
    
2.  _**⭐ 流式数据不知何时、下发怎样的数据：**_A 流的数据到达后，如果 B 流的数据永远不到，那么 A 流的数据在什么时候以及是否要填充一个 null 值下发下去（数据时效问题）
    

从上面两个问题也可以得出大数据中的 2 个核心问题：

1.  _**⭐ 数据质量问题**_
    
2.  _**⭐ 数据时效性问题**_
    

> 注意：
> 
> 博主将上文中的批式计算中的 "全集" 用引号括了起来，**是因为离线这个全集也不是真正的全集**。
> 
> 以天分区表为例，我们在离线计算中常常会遇到数据漂移问题，那么在做数据关联时，由于数据漂移的问题也可能导致有些数据关联不上，所以这个全集也是有数据质量问题的！
> 
> 而实时计算中，数据流都是无界的，反而不会存在这种数据质量问题！
> 
> 这里只是给大家引出博主的这个观点，大家不必细究细节，因为即使批式计算中有少量的数据漂移问题，这点误差基本对业务也没有什么影响。

针对上面的几个问题，博主结合小伙伴萌的意见得出以下的解决方案。

4. 带大家看一遍本文思路
=============

我们在看解决方案之前看一下博主下文在阐述每一种解决方案时的讲述思路。

1.  _**解决方案说明：**_说明每一种解决方案的思路以及这个解决方案是怎么解决上一节说的流式计算的问题的
    
2.  _**解决方案 Flink API：**_说明每一种解决方案，哪种 Flink API 支持以及 Flink API 的使用方法、案例
    
3.  _**解决方案的特点：**_然后说明每一种解决方案在数据质量、时效性上面的特点
    
4.  _**解决方案的适用场景：**_举例说明给每一种解决方案的适用场景
    

5.Flink Join 解决方案：Flink Window Join
===================================

5.1. 解决方案说明
-----------

_**Flink Window Join。**_就是将两条流的数据从无界数据变为有界数据，即划分出时间窗口，然后将同一时间窗口内的两条流的数据做 Join（这里的时间窗口支持 Tumbling、Sliding、Session）。

![](https://mmbiz.qpic.cn/mmbiz_png/DODKOLcDkD1ibow93L6iahP7LhR5YegibuHGTDHTZajgyaYeQ7libRY2TjHgvncoh8OiafacuFfSkbVlskKFMPibLggw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/DODKOLcDkD1ibow93L6iahP7LhR5YegibuHawsaicmIwyoCTz06X2UBr2sowEFw9B17xstr0DRzib1kekmeicHYTjmog/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/DODKOLcDkD1ibow93L6iahP7LhR5YegibuHiaYl5UmQicBBeTSTibGKPwmveyhxAtrkCIPAWLldfw5gUWbLsge9tBh0w/640?wx_fmt=png)

那么该方案怎么解决第 3 节说的两个问题呢？

1.  _**⭐ 流式数据到达计算引擎的时间不一定：**_数据已经被划分为窗口，无界数据变为有界数据，就和离线批处理的方式一样了，两个窗口的数据简单的进行关联即可
    
2.  _**⭐ 流式数据不知何时、下发怎样的数据：**_窗口结束就把数据下发下去，关联到的数据就下发 [A, B]，没有关联到的数据取决于是否是 outer join 然后进行数据下发
    

5.2. 解决方案 Flink API
-------------------

上面这种解决方案目前支持 Flink DataStream API、SQL API 两种。案例如下：

1.  _**⭐ DataStream API：**_
    

```
flinkEnv.env()
    // A 流
    .addSource(new SourceFunction<Object>() {
        @Override
        public void run(SourceContext<Object> ctx) throws Exception {
            
        }
    
        @Override
        public void cancel() {
    
        }
    })
    // B 流
    .join(flinkEnv.env().addSource(new SourceFunction<Object>() {
        @Override
        public void run(SourceContext<Object> ctx) throws Exception {
            
        }
    
        @Override
        public void cancel() {
    
        }
    }))
    // A 流的 keyby 条件
    .where(new KeySelector<Object, Object>() {
        @Override
        public Object getKey(Object value) throws Exception {
            return null;
        }
    })
    // B 流的 keyby 条件
    .equalTo(new KeySelector<Object, Object>() {
        @Override
        public Object getKey(Object value) throws Exception {
            return null;
        }
    })
    // 开窗口
    .window(TumblingEventTimeWindows.of(Time.seconds(60)))
    // 窗口中关联到的数据的处理逻辑
    .apply(new JoinFunction<Object, Object, Object>() {
        @Override
        public Object join(Object first, Object second) throws Exception {
            return null;
        }
    });
```

上述解决方案只支持 inner join，即窗口内能关联到的才会下发，关联不到的则直接丢掉。

如果你想实现 window 上的 outer join，可以使用 coGroup 算子，案例如下：

```
public class CogroupFunctionDemo02 {

    public static void main(String[] args) throws Exception {
        final StreamExecutionEnvironment env=StreamExecutionEnvironment.getExecutionEnvironment();

        // A 流
        DataStream<Tuple2<String,String>> input1=env.socketTextStream("",9002)
                .map(new MapFunction<String, Tuple2<String,String>>() {

                    @Override
                    public Tuple2<String,String> map(String s) throws Exception {

                        return Tuple2.of(s.split(" ")[0],s.split(" ")[1]);
                    }
                });

        // B 流
        DataStream<Tuple2<String,String>> input2=env.socketTextStream("",9001)
                .map(new MapFunction<String, Tuple2<String,String>>() {

                    @Override
                    public Tuple2<String,String> map(String s) throws Exception {

                        return Tuple2.of(s.split(" ")[0],s.split(" ")[1]);
                    }
                });

        // A 流关联 B 流
        input1.coGroup(input2)
                // A 流的 keyby 条件
                .where(new KeySelector<Tuple2<String,String>, Object>() {

                    @Override
                    public Object getKey(Tuple2<String, String> value) throws Exception {
                        return value.f0;
                    }
                }).equalTo(new KeySelector<Tuple2<String,String>, Object>() {
                // B 流的 keyby 条件

            @Override
            public Object getKey(Tuple2<String, String> value) throws Exception {
                return value.f0;
            }
        })
                // 窗口
                .window(ProcessingTimeSessionWindows.withGap(Time.seconds(3)))
                .apply(new CoGroupFunction<Tuple2<String,String>, Tuple2<String,String>, Object>() {

                // 可以自定义实现 A 流和 B 流在关联不到时的输出数据格式

                    @Override
                    public void coGroup(Iterable<Tuple2<String, String>> iterable, Iterable<Tuple2<String, String>> iterable1, Collector<Object> collector) throws Exception {
                        StringBuffer buffer=new StringBuffer();
                        buffer.append("DataStream frist:\n");
                        for(Tuple2<String,String> value:iterable){
                            buffer.append(value.f0+"=>"+value.f1+"\n");
                        }
                        buffer.append("DataStream second:\n");
                        for(Tuple2<String,String> value:iterable1){
                            buffer.append(value.f0+"=>"+value.f1+"\n");
                        }
                        collector.collect(buffer.toString());
                    }
                }).print();

        env.execute();
    }
}
```

或者你还可以使用 connect 算子自定义各种关联操作（connect 算子相比 join、coGroup 算子灵活很多）：

```
// (userEvent, userId)
KeyedStream<UserEvent, String> customerUserEventStream = env
        .addSource(kafkaUserEventSource)
        .assignTimestampsAndWatermarks(new CustomWatermarkExtractor(Time.hours(24)))
        .keyBy(new KeySelector<UserEvent, String>() {
            @Override
            public String getKey(UserEvent userEvent) throws Exception {
                return userEvent.getUserId();
            }
        });
//customerUserEventStream.print();

final BroadcastStream<Config> configBroadcastStream = env
        .addSource(kafkaConfigEventSource)
        .broadcast(configStateDescriptor);

final FlinkKafkaProducer010 kafkaProducer = new FlinkKafkaProducer010<EvaluatedResult>(
        params.get(OUTPUT_TOPIC),
        new EvaluatedResultSerializationSchema(),
        producerProps);

DataStream<EvaluatedResult> connectedStream = customerUserEventStream
        .connect(configBroadcastStream)
        .process(new ConnectedBroadcastProcessFuntion());
```

2.  _**⭐ SQL API（Flink 1.14 版本 Window TVF 中支持）：**_
    

```
SELECT 
    L.num as L_Num
    , L.id as L_Id
    , R.num as R_Num
    , R.id as R_Id
    , L.window_start
    , L.window_end
FROM (
    SELECT * 
    FROM TABLE(TUMBLE(TABLE LeftTable, DESCRIPTOR(row_time), INTERVAL '5' MINUTES))
) L
FULL JOIN (
    SELECT * 
    FROM TABLE(TUMBLE(TABLE RightTable, DESCRIPTOR(row_time), INTERVAL '5' MINUTES))
) R
ON L.num = R.num 
AND L.window_start = R.window_start 
AND L.window_end = R.window_end;
```

5.3. 解决方案的特点
------------

1.  _**⭐ 产出数据质量：低**_
    
2.  _**⭐ 产出数据时效性：中**_
    

当我们的窗口大小划分的越细时，在窗口边缘关联不上的数据就会越多，数据质量就越差。窗口大小划分的越宽时，窗口内关联上的数据就会越多，数据质量越好，但是产出时效性就会越差。所以小伙伴萌在使用时要注意取舍。

举个例子：以曝光关联点击来说，如果我们划分的时间窗口为 1 分钟，那么一旦出现曝光在 0:59，点击在 1:01 的情况，就会关联不上，当我们的划分的时间窗口 1 小时时，只有在每个小时的边界处的数据才会出现关联不上的情况。

5.4. 解决方案的适用场景
--------------

该种解决方案适用于可以评估出窗口内的关联率高的场景，如果窗口内关联率不高则不建议使用。

注意：这种方案由于上面说到的数据质量和时效性问题在实际生产环境中很少使用。

6.Flink Join 解决方案：Flink Interval Join
=====================================

6.1. 解决方案说明
-----------

_**Flink Interval Join。**_其也是将两条流的数据从无界数据变为有界数据，但是这里的有界和上节说到的 Flink Window Join 的有界的概念是不一样的，这里的有界是指两条流之间的有界。

以 A 流 join B 流举例，interval join 可以让 A 流可以关联 B 流一段时间区间内的数据，比如 A 流关联 B 流前后 5 分钟的数据。

![](https://mmbiz.qpic.cn/mmbiz_png/DODKOLcDkD1ibow93L6iahP7LhR5YegibuHTdzAGp11SngYUwkf2JvxVQicWI9OPEM4yJ9T8CyJtNsn2cqWZ5CCRFQ/640?wx_fmt=png)

1

那么该方案怎么解决第 3 节说的两个问题呢？

1.  _**⭐ 流式数据到达计算引擎的时间不一定：**_数据已经被划分为窗口，无界数据变为有界数据，就和离线批处理的方式一样了，两个窗口的数据简单的进行关联即可
    
2.  _**⭐ 流式数据不知何时、下发怎样的数据：**_窗口结束（这里的窗口结束是指 interval 区间结束，区间的结束是利用 watermark 来判断的）就把数据下发下去，关联到的数据就下发 [A, B]，没有关联到的数据取决于是否是 outer join 然后进行数据下发
    

6.2. 解决方案 Flink API
-------------------

上面这种解决方案目前支持 Flink DataStream API 和 SQL API 两种。案例如下：

1.  _**⭐ DataStream API：**_
    

```
clickRecordStream
  .keyBy(record -> record.getMerchandiseId())
  .intervalJoin(orderRecordStream.keyBy(record -> record.getMerchandiseId()))
  // 定义 interval 的时间区间
  .between(Time.seconds(-30), Time.seconds(30))
  .process(new ProcessJoinFunction<AnalyticsAccessLogRecord, OrderDoneLogRecord, String>() {
    @Override
    public void processElement(AnalyticsAccessLogRecord accessRecord, OrderDoneLogRecord orderRecord, Context context, Collector<String> collector) throws Exception {
      collector.collect(StringUtils.join(Arrays.asList(
        accessRecord.getMerchandiseId(),
        orderRecord.getPrice(),
        orderRecord.getCouponMoney(),
        orderRecord.getRebateAmount()
      ), '\t'));
    }
  })
  .print();
```

2.  _**⭐ SQL API：**_
    

```
CREATE TABLE show_log_table (
     log_id BIGINT,
     show_params STRING,
     row_time AS cast(CURRENT_TIMESTAMP as timestamp(3)),
     WATERMARK FOR row_time AS row_time
 ) WITH (
   'connector' = 'datagen',
   'rows-per-second' = '1',
   'fields.show_params.length' = '1',
   'fields.log_id.min' = '1',
   'fields.log_id.max' = '10'
 );
 
 CREATE TABLE click_log_table (
     log_id BIGINT,
     click_params STRING,
     row_time AS cast(CURRENT_TIMESTAMP as timestamp(3)),
     WATERMARK FOR row_time AS row_time
 )
 WITH (
   'connector' = 'datagen',
   'rows-per-second' = '1',
   'fields.click_params.length' = '1',
   'fields.log_id.min' = '1',
   'fields.log_id.max' = '10'
 );
 
 CREATE TABLE sink_table (
     s_id BIGINT,
     s_params STRING,
     c_id BIGINT,
     c_params STRING
 ) WITH (
   'connector' = 'print'
 );
 
 INSERT INTO sink_table
 SELECT
     show_log_table.log_id as s_id,
     show_log_table.show_params as s_params,
     click_log_table.log_id as c_id,
     click_log_table.click_params as c_params
 FROM show_log_table FULL JOIN click_log_table ON show_log_table.log_id = click_log_table.log_id
 AND show_log_table.row_time BETWEEN click_log_table.row_time - INTERVAL '5' SECOND AND click_log_table.row_time
```

6.3. 解决方案的特点
------------

1.  _**⭐ 产出数据质量：中**_
    
2.  _**⭐ 产出数据时效性：中**_
    

interval join 的方案比 window join 方案在数据质量上好很多，但是其也是存在 join 不到的情况的。并且如果为 outer join 的话，outer 一测的流数据需要要等到区间结束才能下发。

6.4. 解决方案的适用场景
--------------

该种解决方案适用于两条流之间可以明确评估出相互延迟的时间是多久的，这里我们可以使用离线数据进行评估，使用离线数据的两条流的时间戳做差得到一个分布区间。

比如在 A 流和 B 流时间戳相差在 1min 之内的有 95%，在 1-4 min 之内的有 4.5%，则我们就可以认为两条流数据时间相差在 4 min 之内的有 99.5%，这时我们将上下界设置为 4min 就是一个能保障 0.5% 误差的合理区间。

注意：这种方案在生产环境中还是比较常用的。

7.Flink Join 解决方案：Flink Regular Join
====================================

7.1. 解决方案说明
-----------

_**Flink Regular Join。**_上面两节说的两种 Join 都是基于划分窗口，将无界数据变为有界数据进行关联机制，但是本节说的 regular join 则还是基于无界数据进行关联。

以 A 流 left join B 流举例，A 流数据到来之后，直接去尝试关联 B 流数据。

1.  ⭐ 如果关联到了则直接下发关联到的数据
    
2.  ⭐ 如果没有关联到则也直接下发没有关联到的数据，后续 B 流中的数据到来之后，会把之前下发下去的没有关联到数据撤回，然后把关联到的数据数据进行下发。由此可以看出这是基于 Flink SQL 的 retract 机制，则也就说明了其目前只支持 Flink SQL。
    

![](https://mmbiz.qpic.cn/mmbiz_png/DODKOLcDkD1ibow93L6iahP7LhR5YegibuHobZBhvl3yvhBlUwkvmheHb1aEsibeWle0T7cYDN1pRA9M1bfW4EmumQ/640?wx_fmt=png)

5

那么该方案怎么解决第 3 节说的两个问题呢？

1.  ⭐ 流式数据到达计算引擎的时间不一定：两条流的数据会尝试关联，能关联到直接下发，关联不到先下发一个目前的结果数据
    
2.  ⭐ 流式数据不知何时、下发怎样的数据：两条流的数据会尝试关联，能关联到直接下发，关联不到先下发一个目前的结果数据
    

博主认为这是目前最好的一种数据关联方式。

7.2. 解决方案 Flink API
-------------------

上面这种解决方案目前只支持 SQL API。案例如下：

```
CREATE TABLE show_log_table (
    log_id BIGINT,
    show_params STRING
) WITH (
  'connector' = 'datagen',
  'rows-per-second' = '1',
  'fields.show_params.length' = '3',
  'fields.log_id.min' = '1',
  'fields.log_id.max' = '10'
);

CREATE TABLE click_log_table (
  log_id BIGINT,
  click_params     STRING
)
WITH (
  'connector' = 'datagen',
  'rows-per-second' = '1',
  'fields.click_params.length' = '3',
  'fields.log_id.min' = '1',
  'fields.log_id.max' = '10'
);

CREATE TABLE sink_table (
    s_id BIGINT,
    s_params STRING,
    c_id BIGINT,
    c_params STRING
) WITH (
  'connector' = 'print'
);

INSERT INTO sink_table
SELECT
    show_log_table.log_id as s_id,
    show_log_table.show_params as s_params,
    click_log_table.log_id as c_id,
    click_log_table.click_params as c_params
FROM show_log_table
LEFT JOIN click_log_table ON show_log_table.log_id = click_log_table.log_id;
```

7.3. 解决方案的特点
------------

1.  _**⭐ 产出数据质量：高**_
    
2.  _**⭐ 产出数据时效性：高**_
    

数据质量和时效性高的原因都是因为 regular join 会保障目前 Flink 任务已经接收到的数据中能关联的一定是关联上的，即使关联不上，数据也会下发，完完全全保障了当前数据的客观性和时效性。

7.4. 解决方案的适用场景
--------------

该种解决方案虽然是目前在产出质量、时效性上最好的一种解决方案，但是在实际场景中使用时，也存在一些问题：

1.  ⭐ 基于 retract 机制，所有的数据都会存储在 state 中以判断能否关联到，所以我们要设置合理的 state ttl 来避免大 state 问题导致的任务不稳定
    
2.  ⭐ 基于 retract 机制，所以在数据发生更新时，会下发回撤数据、最新数据 2 条消息，当我们的关联层级越多，则下发消息量的也会放大
    
3.  ⭐ sink 组件要支持 retract，我们不要忘了最终数据是要提供数据服务给需求方进行使用的，所以我们最终写入的数据组件也需要支持 retract，比如 MySQL。如果写入的是 Kafka，则下游消费这个 Kafka 的引擎也需要支持回撤 \ 更新机制。
    

8. 上述 3 种解决方案各有优劣，有没有什么共性的问题可以优化？
=================================

针对上面 3 节说到的 Flink Join 的方案，各自都有一些优势和劣势存在。

但是我们可以发现，无论是哪一种 Join 方案，Join 的前提都是将 A 流和 B 流的数据先存储在状态中，然后再进行关联。

即在实际生产中使用时常常会碰到的问题就是：大状态的问题。

关于大状态问题业界常见两种解决思路：

1.  _**⭐ 减少状态大小：**_在 Flink Join 中的可以想到的优化措施就是减少 state key 的数量。在未优化之前 A 流和 B 流的数据往往是存储在单独的两个 State 实例中的，那么我们的优化思路就是将同 Key 的数据放在一起进行存储，一个 key 的数据只需要存储一份，减少了 key 的数量
    
2.  _**⭐ 转移状态至外存：**_大 State 会导致 Flink 任务不稳定，那么我们就将 State 存储在外存中，让 Flink 任务轻量化，比如将数据存储在 Redis 中，A 流和 B 流中相同 key 的数据共同维护在一个 Redis 的 hashmap 中，以供相互进行关联
    

接下来看看这两种方案实际需要怎样落地。讲述思路也是按照以下几点进行阐述：

1.  _**⭐ 优化方案说明**_
    
2.  _**⭐ 优化方案 Flink API**_
    
3.  _**⭐ 优化方案的特点**_
    
4.  _**⭐ 优化方案的适用场景**_
    

9.Flink Join 优化方案：同 key 共享 State
================================

9.1. 优化方案说明
-----------

将两条流的数据使用 union、connect 算子合并在一起，然后使用一个共享的 state 进行处理。

![](https://mmbiz.qpic.cn/mmbiz_png/DODKOLcDkD1ibow93L6iahP7LhR5YegibuHibp9oOroibtvb1bIbIOdpbibjUePMEpmDcRA8vEYCTEd86yxrfTQqD1tA/640?wx_fmt=png)

7

9.2. 优化方案 Flink API
-------------------

上面这种优化方案建议使用 DataStream API。案例如下：

```
FlinkEnv flinkEnv = FlinkEnvUtils.getStreamTableEnv(args);

flinkEnv.env().setParallelism(1);

flinkEnv.env()
    .addSource(new SourceFunction<Object>() {
        @Override
        public void run(SourceContext<Object> ctx) throws Exception {

        }

        @Override
        public void cancel() {

        }
    })
    .keyBy(new KeySelector<Object, Object>() {
        @Override
        public Object getKey(Object value) throws Exception {
            return null;
        }
    })
    .connect(flinkEnv.env().addSource(new SourceFunction<Object>() {
        @Override
        public void run(SourceContext<Object> ctx) throws Exception {

        }

        @Override
        public void cancel() {

        }
    }).keyBy(new KeySelector<Object, Object>() {
        @Override
        public Object getKey(Object value) throws Exception {
            return null;
        }
    }))
    // 左右两条流的数据
    .process(new KeyedCoProcessFunction<Object, Object, Object, Object>() {
        // 两条流的数据共享一个 mapstate 进行处理
        private transient MapState<String, String> mapState;

        @Override
        public void open(Configuration parameters) throws Exception {
            super.open(parameters);
            
            this.mapState = getRuntimeContext().getMapState(new MapStateDescriptor<String, String>("a", String.class, String.class));
        }

        @Override
        public void processElement1(Object value, Context ctx, Collector<Object> out) throws Exception {
            
        }

        @Override
        public void processElement2(Object value, Context ctx, Collector<Object> out) throws Exception {

        }
    })
    .print();
```

9.3. 优化方案的特点
------------

在此种优化方案下，我们可以自定义：

1.  _**⭐ state 的过期方式**_
    
2.  _**⭐ 左右两条流的数据的 state 中的存储方式**_
    
3.  _**⭐ 左右两条流数据在关联不到对方的情况下是否要输出到下游、输出什么样的数据到下游的方式**_
    

9.4. 优化方案的适用场景
--------------

该种解决方案适用于可以做 state 清理的场景，比如在曝光关联点击的情况下，如果我们能明确一次曝光只有一次点击的话，只要这条曝光或者点击被关联到过，那么我们就可以在 `KeyedCoProcessFunction` 中自定义逻辑将已经被关联过得曝光、点击的 state 数据进行删除，以减小 state，减轻任务压力。

10.Flink Join 优化方案：同 key 共享 State
=================================

10.1. 外存 State 之 Redis
----------------------

此种方案就是完全不使用 Flink 的 state，直接将来的数据存储到 Redis 中进行维护，A 流的数据过来之后，去 Redis 中找 B 流的数据，B 流的数据过来之后，去 Redis 中找 A 流的数据。

![](https://mmbiz.qpic.cn/mmbiz_png/DODKOLcDkD1ibow93L6iahP7LhR5YegibuHchWsjvyD2uM5CMMZZuMKTWp7XeOJgS0oLa1iaQOgkOJ7E93IbTYwJ4w/640?wx_fmt=png)

6

10.2. 优化方案 Flink API
--------------------

比如常用的 Redis HashMap 结构等。

10.3. 优化方案的特点
-------------

在此种优化方案下：

1.  _**⭐ Flink 是轻量级的，不需要担心大状态问题**_
    
2.  _**⭐ 外存容量可以随时扩容，理论上多大的 state 都可以存储，适合某些不能对 state 做清除的场景**_
    

10.4. 优化方案的适用场景
---------------

1.  ⭐ 某些金融公司内的关联，state 是不能被清理的，比如存储了借款信息之后，这些信息后续还是可能被修改的。所以这种场景下需要存储全量的 state
    
2.  ⭐ 比如某东的交易场景下，A 流的数据和 B 流的数据生成的时间相差很大的，比如 A 流数据先生产之后，B 流数据在 1 天之后才会被生产出来，那么这种场景就非常适合使用 redis 存储 A 流数据，后续提供给 B 流使用
    

11. 小结
======

当然上述的解决方案和优化方案有的之间是可以相互结合的。小伙伴萌可以结合实际情况进行使用。

比如 _**interval join + redis state**_ 存储。

1.  ⭐ interval join 解决两条流之间一段时间区间内的数据关联，可以保障 Flink 中的 state 数据量小，任务稳定。
    
2.  ⭐ redis 解决时间区间之外的数据关联问题，保障数据准确性。
    

往期推荐

[

爆肝 1 年，18w 字 Flink SQL 手册，横空出世 ！！！(建议收藏)



](https://mp.weixin.qq.com/s?__biz=MzkxNjA1MzM5OQ==&mid=2247491845&idx=1&sn=163341d868de941e902041fea7148ef1&chksm=c1576dfdf620e4eb2cfcdaf056b138d920ac43a78e1efe75a7d25cabcdfc70cc36aeb3dd279d&scene=21#wechat_redirect)

[

flink sql 知其所以然（十三）：流 join 很难嘛？？？（下）



](https://mp.weixin.qq.com/s?__biz=MzkxNjA1MzM5OQ==&mid=2247489658&idx=1&sn=6004d88772d473d4f5f446b8ffd3e14f&chksm=c1549482f6231d94a2c2841ff2b1ba840573dfd5acb6360ba84b9205cebd5a4a7c7f0aba61fc&scene=21#wechat_redirect)

[

flink sql 知其所以然（十二）：流 join 很难嘛？？？（上）



](https://mp.weixin.qq.com/s?__biz=MzkxNjA1MzM5OQ==&mid=2247489633&idx=1&sn=24b418a8192116306eb3aab00ff24600&chksm=c1549499f6231d8ff40cdacd0504a21e605c07ba37fcfb4f5877523bac727e7955702882d7a2&scene=21#wechat_redirect)

[

flink sql 知其所以然（十一）：去重不仅仅有 count distinct 还有强大的 deduplication



](https://mp.weixin.qq.com/s?__biz=MzkxNjA1MzM5OQ==&mid=2247489624&idx=1&sn=2738ec774dad30ae69b475dce45bfe4a&chksm=c15494a0f6231db6479fc45d18a3bdd69472d60240e9227edb4424cac420ed9f06381fd52a28&scene=21#wechat_redirect)

[

flink sql 知其所以然（十）：大家都用 cumulate window 计算累计指标啦



](https://mp.weixin.qq.com/s?__biz=MzkxNjA1MzM5OQ==&mid=2247489554&idx=1&sn=275ca7bd853a762912f43bc51ef5c65f&chksm=c15494eaf6231dfca785e01632b8194db1817fb326c2698098e3eaeae430448596d73bc4a9e4&scene=21#wechat_redirect)

[当我们在做流批一体时，我们在做什么？](https://mp.weixin.qq.com/s?__biz=MzkxNjA1MzM5OQ==&mid=2247489496&idx=1&sn=016e580c5932b232005ff1d5e345588a&chksm=c1549b20f6231236571444c621f5c94dedc06daad7db0fa662674e544424e4a0d0fb16df3168&scene=21#wechat_redirect)

![](https://mmbiz.qpic.cn/mmbiz_png/eAM2voboIPMgiatDaS2mLcIjv8T03zB7u0Z8LFwPNttaw5qbm1m0ic21wibOHA1qqaQQ1S7sla5A1z8THeSD1MjFQ/640?wx_fmt=png)

—END—

你好，我是老羊，一个互联网大厂实时数据开发工程师。

在公众号发表的 18w 字、132 案例、48 图 实时计算入门**《Flink SQL 成神之路》**、实时计算进阶**《Flink SQL 知其所以然》**、实时计算面试**《Flink 对线面试官》**系列深受读者好评，关注公众号发送【Flink SQL】限时免费领取《Flink SQL 成神之路》PDF。

我每周至少更新一篇原创，不仅分享大数据相关资讯，更多关注于思维模式的升级，以及程序人生的感悟，希望每一篇推送都能给到你不一样的启发。我正前行在实现自己目标的路上，**下方关注后可以加我微信交流。愿你所往之地皆为热土，愿你所遇之人皆为挚友。**

![](https://mmbiz.qpic.cn/mmbiz_jpg/DODKOLcDkD1vbQXT4HBrNQ1km0OU1GrRGia0RkCAOy14lLxKae1Bn9G5zJ1VqVvibRiczRyoziaIVd0yZxCMbU0tGw/640?wx_fmt=jpeg)

长按上方扫码二维码，加我微信

![](https://mmbiz.qpic.cn/mmbiz/cZV2hRpuAPia3RFX6Mvw06kePJ7HbmI7b35o17yNJx4WHYPSQj280IElEicRPq2CviaJe8fjL2AeadmIjARqVZWnw/640?wx_fmt=jpeg)  

👆点击关注｜设为星标｜干货速递👆

动动小手，让更多需要的人看到~

![](https://mmbiz.qpic.cn/mmbiz_gif/I0wRNtcLDEfq42GGpPpCfpAUGFrQ9pSeyR1yB1uvpUm4ia6A3eYdKlibggxeZjNj5M3WZabwicv5ojMv88gwicfQOw/640?wx_fmt=gif)