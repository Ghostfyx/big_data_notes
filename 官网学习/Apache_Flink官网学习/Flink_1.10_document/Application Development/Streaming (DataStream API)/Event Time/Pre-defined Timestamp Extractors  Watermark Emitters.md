# Pre-defined Timestamp Extractors / Watermark Emitters

如[timestamps and watermark handling](https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/event_timestamps_watermarks.html)所述，Flink提供了抽象，允许程序员分配自己的时间戳并发出水印。 根据使用情况，可以通过实现AssignerWithPeriodicWatermarks和AssignerWithPunctuatedWatermarks接口之一来做到这一点。 简而言之，第一个会定期发出水印，而第二个会根据传入记录的某些属性来发出水印，例如：每当在流中遇到特殊元素时。

为了进一步简化此类任务的编程工作，Flink附带了一些预先实现的时间戳分配器。

## 1. 递增时间戳分配器

定期生成水印的最简单的情况是给定源任务的时间戳以升序出现的情况。 在这种情况下，当前时间戳始终可以充当水印，因为没有更早的时间戳会到达。

请注意，每个并行数据源任务的时间戳都仅递增。 例如，如果在一个特定的设置中，一个并行数据源实例读取一个Kafka分区，则只需要在每个Kafka分区内将时间戳记递增。 每当对并行流进行混洗，合并，连接或合并时，Flink的水印合并机制将生成正确的水印。

```java
DataStream<MyEvent> stream = ...

DataStream<MyEvent> withTimestampsAndWatermarks =
    stream.assignTimestampsAndWatermarks(new AscendingTimestampExtractor<MyEvent>() {

        @Override
        public long extractAscendingTimestamp(MyEvent element) {
            return element.getCreationTime();
        }
});
```

## 2. 固定延迟时间分配器

周期性水印生成的另一个示例是水印在流中看到的最大（事件时间）时间戳落后固定时间量的情况。这种情况涵盖了事先知道流中可能遇到的最大延迟的场景，例如当创建包含带有时间戳的元素的自定义源时，该时间戳会在固定的时间段内传播以进行测试。对于这些情况，Flink提供了BoundedOutOfOrdernessTimestampExtractor，它以maxOutOfOrderness作为参数，即在计算给定窗口的最终结果时允许元素延迟到被忽略之前的最长时间。延迟对应于t-t_w的结果，其中t是元素的（事件时间）时间戳，而t_w是先前水印的时间戳。如果延迟> 0，则将元素视为延迟，并且默认情况下，在为其相应窗口计算作业结果时将忽略该元素。

```java
DataStream<MyEvent> stream = ...

DataStream<MyEvent> withTimestampsAndWatermarks =
    stream.assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<MyEvent>(Time.seconds(10)) {

        @Override
        public long extractTimestamp(MyEvent element) {
            return element.getCreationTime();
        }
});
```

