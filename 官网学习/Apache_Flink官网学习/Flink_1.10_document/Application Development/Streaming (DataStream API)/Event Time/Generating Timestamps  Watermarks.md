# Generating Timestamps / Watermarks

为了处理事件时间，流式传输程序需要相应地设置时间特征。

```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
```

## 1. Assigning Timestamps

为了使用事件时间，Flink需要知道事件的时间戳，这意味着流中的每个元素都需要分配其事件时间戳。 这通常是通过从元素的某个字段访问/提取时间戳来完成的。

时间戳分配伴随着生成水印，水印告诉系统事件时间的进展。有两种方式分配时间戳和生产水印：

1. 直接在数据源生成；
2. 通过时间戳分配器/水印生成器：在Flink中，时间戳分配器还定义要发送的水印

注意：时间戳和水印都被指定为自1970-01-01T00：00：00Z的Java纪元以来的毫秒。

## 2. Source Functions with Timestamps and Watermarks

Stream Source可以将时间戳直接分配给它们产生的元素，并且还可以发出水印。 完成此操作后，无需时间戳分配器。 请注意，如果使用时间戳分配器，则源所提供的任何时间戳和水印都将被覆盖。

为了在DataSource处直接分配时间戳，source必须在sourceContext上用`collectWithTimestamp(...)`方法，为了生成watermarks，source必须调用`emitWatermark(Watermark)`方法。

下面是一个简单的示例(非检查点)source，该源分配时间戳并生成水印：

```java
@Override
public void run(SourceContext<MyType> ctx) throws Exception {
	while (/* condition */) {
		MyType next = getNext();
		ctx.collectWithTimestamp(next, next.getEventTimestamp());

		if (next.hasWatermarkTime()) {
			ctx.emitWatermark(new Watermark(next.getWatermarkTime()));
		}
	}
}
```

## 3. Timestamp Assigners / Watermark Generators

Timestamp Assigners获取流并产生带有时间戳记的元素和水印的新流。 如果原始流已经具有时间戳和/或水印，则时间戳分配器将覆盖它们。

时间戳记分配器通常在数据源之后立即指定，但并非严格要求这样做。 例如，一种常见的模式是在时间戳分配器之前解析(MapFunction)和筛选器(FilterFunction)。 无论如何，都需要在事件时间的第一个操作(例如第一个窗口操作)之前指定时间戳分配器。 作为一种特殊情况，当使用Kafka作为流作业的源时，Flink允许在源(或使用者)本身内部指定时间戳分配器/水印发射器。

```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

DataStream<MyEvent> stream = env.readFile(
        myFormat, myFilePath, FileProcessingMode.PROCESS_CONTINUOUSLY, 100,
        FilePathFilter.createDefaultFilter(), typeInfo);

DataStream<MyEvent> withTimestampsAndWatermarks = stream
        .filter( event -> event.severity() == WARNING )
        .assignTimestampsAndWatermarks(new MyTimestampsAndWatermarks());

withTimestampsAndWatermarks
        .keyBy( (event) -> event.getGroup() )
        .timeWindow(Time.seconds(10))
        .reduce( (a, b) -> a.add(b) )
        .addSink(...);
```

### 3.1 使用周期性水印

`AssignerWithPeriodicWatermarks`定期分配时间戳和生成水印(可能基于流数据元或纯粹基于处理时间)。

通过`ExecutionConfig.setAutoWatermarkInterval(...)`定义生成水印的时间间隔(每n毫秒)。 每次都会调用分配者的`getCurrentWatermark()`方法，如果返回的水印非空且大于前一个水印，则将发出新的水印。

下面展示了Flink使用时间戳分配器定期生成水印，注意：Flink附带`BoundedOutOfOrdernessTimestampExtractor`

```java
/**
 * This generator generates watermarks assuming that elements arrive out of order,
 * but only to a certain degree. The latest elements for a certain timestamp t will arrive
 * at most n milliseconds after the earliest elements for timestamp t.
 */
public class BoundedOutOfOrdernessGenerator implements AssignerWithPeriodicWatermarks<MyEvent> {

    private final long maxOutOfOrderness = 3500; // 3.5 seconds

    private long currentMaxTimestamp;

    @Override
    public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
        long timestamp = element.getCreationTime();
        currentMaxTimestamp = Math.max(timestamp, currentMaxTimestamp);
        return timestamp;
    }

    @Override
    public Watermark getCurrentWatermark() {
        // return the watermark as current highest timestamp minus the out-of-orderness bound
        return new Watermark(currentMaxTimestamp - maxOutOfOrderness);
    }
}

/**
 * This generator generates watermarks that are lagging behind processing time by a fixed amount.
 * It assumes that elements arrive in Flink after a bounded delay.
 */
public class TimeLagWatermarkGenerator implements AssignerWithPeriodicWatermarks<MyEvent> {

	private final long maxTimeLag = 5000; // 5 seconds

	@Override
	public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
		return element.getCreationTime();
	}

	@Override
	public Watermark getCurrentWatermark() {
		// return the watermark as current time minus the maximum time lag
		return new Watermark(System.currentTimeMillis() - maxTimeLag);
	}
}
```

### 3.2 使用间断水印

要在特定事件表明可能生成新的水印时生成水印，请使用AssignerWithPunctuatedWatermarks。 对于此类，Flink将首先调用extractTimestamp(...)方法为该元素分配时间戳，然后立即在该元素上调用checkAndGetNextWatermark(...)方法。

将checkAndGetNextWatermark(...)方法传递给extractTimestamp(...)方法中分配的时间戳，并可以决定是否要生成水印。 每当checkAndGetNextWatermark(...)方法返回非空水印，并且该水印大于最新的先前水印时，就会发出新水印。

```java
public class PunctuatedAssigner implements AssignerWithPunctuatedWatermarks<MyEvent> {

	@Override
	public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
		return element.getCreationTime();
	}

	@Override
	public Watermark checkAndGetNextWatermark(MyEvent lastElement, long extractedTimestamp) {
		return lastElement.hasWatermarkMarker() ? new Watermark(extractedTimestamp) : null;
	}
}
```

注意：可以在每个事件上生成水印。 但是，由于每个水印都会在下游引起一些计算，因此过多的水印会降低性能。

## 4. Timestamps per Kafka Partition

当使用Apache Kafka作为数据源时，每个Kafka分区可能都有一个简单的事件时间模式（升序时间戳或有界乱序）。 但是，当使用来自Kafka的流时，通常会并行使用多个分区，从而将事件与分区进行交错并破坏每个分区的模式(这是Kafka的客户客户端工作方式所固有的)。

在这样的情况下可以使用Flink的Kafka-partition-aware watermark生成器，每个Kafka分区在Kafka消费者内部生成水印。并且每个分区水印的合并方式与在流shuffle上合并水印的方式相同。

例如：如果事件时间戳在Kafka每个分区内严格有效，使用升序水印生成器(AscendingTimestampExtractor)生成每分区水印，将产生完美的整体水印。

下图显示了如何使用per-Kafka分区水印生成，以及在这种情况下水印如何通过流数据流传播。

```java
FlinkKafkaConsumer09<MyType> kafkaSource = new FlinkKafkaConsumer09<>("myTopic", schema, props);
kafkaSource.assignTimestampsAndWatermarks(new AscendingTimestampExtractor<MyType>() {

    @Override
    public long extractAscendingTimestamp(MyType element) {
        return element.eventTimestamp();
    }
});

DataStream<MyType> stream = env.addSource(kafkaSource);
```

![](https://ci.apache.org/projects/flink/flink-docs-release-1.10/fig/parallel_kafka_watermarks.svg)