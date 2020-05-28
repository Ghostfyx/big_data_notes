# Working with State

## 1. Keyed State and Operator State

Flink有两个基本state：Keyed State和Operator State

### 1.1 Keyed State

Keyed State总是与keys相关联，并且只能在KeyedStream的函数和运算符中使用。

可以将Keyed State视为已分区的操作状态。每个键仅具有一个状态分区，每个keyed state逻辑上都绑定到`<parallel-operator-instance, key>`的唯一组合，并且每个键都完全属于keyed操作符的一个并行实例，因此可以简单的视为`<operator, key>`。

Keyed stated被进一步组成Key Groups。Key Groups是Flink可以重新分配Keyed State的原子状态，Key Groups数量与最大并行度完全相同。在执行期间，每个Keyed Operator并行实例都使用一个或者多个键组的键。

### 1.2 Operator State

使用运算符状态(或非键控状态)，每个运算符状态都绑定到一个并行运算符实例。Kakfa Connector是一个使用Flink Operator State很好示例。Kafka Consumer的每一个并行实例都维护一个主题分区和偏移量的映射作为其operator state。

当修改并行度时，Operator State接口支持在Operator并行实例之间重新分配。可以有不同的方案来进行此重新分配。

## 2. Raw State 与 Managed State

Keyed State和Operator State 分别有两种存在形式：*managed* and *raw*.

Managed State 由 Flink 运行时控制的数据结构表示，比如内部的 hash table 或者 RocksDB。 比如 “ValueState”, “ListState” 等。Flink 运行时会对这些状态进行编码并写入 checkpoint。

Row State则保存在算子自己的数据结构中。checkPoint的时候，Flink并不知晓具体内容，仅仅写入一串字节序列到 checkpoint。

所有 datastream 的 function 都可以使用 managed state, 但是 raw state 则只能在实现算子的时候使用。 由于 Flink 可以在修改并发时更好的分发状态数据，并且能够更好的管理内存，因此建议使用 managed state(而不是 raw state)。

**注意** 如果managed state需要定制化的序列化逻辑， 为了后续的兼容性请参考 [相应指南](https://ci.apache.org/projects/flink/flink-docs-release-1.10/zh/dev/stream/state/custom_serialization.html)，Flink 的默认序列化器不需要用户做特殊的处理。

## 3. 使用 Managed Keyed State

managed keyed state 接口提供不同类型状态的访问接口，这些状态都作用于当前输入数据的 key 下。换句话说，这些状态仅可在 `KeyedStream` 上使用，可以通过 `stream.keyBy(...)` 得到 `KeyedStream`.

接下来，介绍不同类型的状态，然后介绍如何使用它们。所有支持的状态类型如下所示：

- `ValueState<T>`: 保存一个可以更新和检索的值（如上所述，每个值都对应到当前的输入数据的 key，因此算子接收到的每个 key 都可能对应一个值）。 这个值可以通过 `update(T)` 进行更新，通过 `T value()` 进行检索。
- `ListState<T>`: 保存一个元素的列表。可以往这个列表中追加数据，并在当前的列表上进行检索。可以通过 `add(T)` 或者 `addAll(List<T>)` 进行添加元素，通过 `Iterable<T> get()` 获得整个列表。还可以通过 `update(List<T>)` 覆盖当前的列表。
- `ReducingState<T>`: 保存一个单值，表示添加到状态的所有值的聚合。接口与 `ListState` 类似，但使用 `add(T)` 增加元素，会使用提供的 `ReduceFunction` 进行聚合。
- `AggregatingState<IN, OUT>`: 保留一个单值，表示添加到状态的所有值的聚合。和 `ReducingState` 相反的是, 聚合类型可能与 添加到状态的元素的类型不同。 接口与 `ListState` 类似，但使用 `add(IN)` 添加的元素会用指定的 `AggregateFunction` 进行聚合。
- `FoldingState<T, ACC>`: 保留一个单值，表示添加到状态的所有值的聚合。 与 `ReducingState` 相反，聚合类型可能与添加到状态的元素类型不同。 接口与 `ListState` 类似，但使用`add（T）`添加的元素会用指定的 `FoldFunction` 折叠成聚合值。
- `MapState<UK, UV>`: 维护了一个映射列表。 你可以添加键值对到状态中，也可以获得反映当前所有映射的迭代器。使用 `put(UK，UV)` 或者 `putAll(Map<UK，UV>)` 添加映射。 使用 `get(UK)` 检索特定 key。 使用 `entries()`，`keys()` 和 `values()` 分别检索映射、键和值的可迭代视图。你还可以通过 `isEmpty()` 来判断是否包含任何键值对。

所有类型的状态还有一个`clear()` 方法，清除当前 key 下的状态数据，也就是当前输入元素的 key。

**注意** `FoldingState` 和 `FoldingStateDescriptor` 从 Flink 1.4 开始就已经被启用，将会在未来被删除。 作为替代请使用 `AggregatingState` 和 `AggregatingStateDescriptor`。

请牢记，这些状态对象仅用于与状态交互。状态本身不一定存储在内存中，还可能在磁盘或其他位置。 另外需要牢记的是从状态中获取的值取决于输入元素所代表的 key。 因此，在不同 key 上调用同一个接口，可能得到不同的值。

**必须创建一个 `StateDescriptor`，才能得到对应的状态句柄**。 这保存了状态名称（正如我们稍后将看到的，你可以创建多个状态，并且它们必须具有唯一的名称以便可以引用它们）， 状态所持有值的类型，并且可能包含用户指定的函数，例如`ReduceFunction`。 根据不同的状态类型，可以创建`ValueStateDescriptor`，`ListStateDescriptor`， `ReducingStateDescriptor`，`FoldingStateDescriptor` 或 `MapStateDescriptor`。

状态通过 `RuntimeContext` 进行访问，因此只能在 *rich functions* 中使用。请参阅[这里](https://ci.apache.org/projects/flink/flink-docs-release-1.10/zh/dev/api_concepts.html#rich-functions)获取相关信息， 但是我们很快也会看到一个例子。`RichFunction` 中 `RuntimeContext` 提供如下方法：

- `ValueState<T> getState(ValueStateDescriptor<T>)`
- `ReducingState<T> getReducingState(ReducingStateDescriptor<T>)`
- `ListState<T> getListState(ListStateDescriptor<T>)`
- `AggregatingState<IN, OUT> getAggregatingState(AggregatingStateDescriptor<IN, ACC, OUT>)`
- `FoldingState<T, ACC> getFoldingState(FoldingStateDescriptor<T, ACC>)`
- `MapState<UK, UV> getMapState(MapStateDescriptor<UK, UV>)`

下面是一个 `FlatMapFunction` 的例子，展示了如何将这些部分组合起来：

```java
public class CountWindowAverage extends RichFlatMapFunction<Tuple2<Long, Long>, Tuple2<Long, Long>> {

    /**
     * The ValueState handle. The first field is the count, the second field a running sum.
     *
     * ValueState存储一个值的状态
     */
    private transient ValueState<Tuple2<Long, Long>> sum;


    @Override
    public void flatMap(Tuple2<Long, Long> value, Collector<Tuple2<Long, Long>> out) throws Exception {
        // access the state value
        Tuple2<Long, Long> currentSum = sum.value();

        // update the count
        currentSum.f0 += 1;

        // add the second field of the input value
        currentSum.f1 += value.f1;

        // update the state
        sum.update(currentSum);

        // if the count reaches 2, emit the average and clear the state
        if (currentSum.f0 >= 2) {
            out.collect(new Tuple2<>(value.f0, currentSum.f1 / currentSum.f0));
            sum.clear();
        }
    }

    @Override
    public void open(Configuration configuration){
        ValueStateDescriptor<Tuple2<Long, Long>> descriptor = new ValueStateDescriptor<>(
                "average", // the state name
                TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}), // type information
                Tuple2.of(0L, 0L)
                ); // default value of the state, if nothing was set
        sum = getRuntimeContext().getState(descriptor);
    }
}
```

这个例子实现了一个简单的计数窗口。 我们把元组的第一个元素当作 key（在示例中都 key 都是 “1”）。 该函数将出现的次数以及总和存储在 “ValueState” 中。 一旦出现次数达到 2，则将平均值发送到下游，并清除状态重新开始。 请注意，我们会为每个不同的 key（元组中第一个元素）保存一个单独的值。

## 4. 状态有效期(TTL)

任何类型的 keyed state 都可以有有效期(TTL)，如果配置了 TTL 且状态值已过期，则会尽最大可能清除对应的值，这会在后面详述。所有状态类型都支持单元素的 TTL。 这意味着列表元素和映射元素将独立到期。

在使用状态 TTL 前，需要先构建一个`StateTtlConfig` 配置对象。 然后把配置传递到 state descriptor 中启用 TTL 功能：

```java
import org.apache.flink.api.common.state.StateTtlConfig;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.api.common.time.Time;

StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
    .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)
    .build();
    
ValueStateDescriptor<String> stateDescriptor = new ValueStateDescriptor<>("text state", String.class);
stateDescriptor.enableTimeToLive(ttlConfig);
```

TTL配置有以下几个选项，newBuilder的第一个参数表示数据的有效期，是必选项。

TTL的更新策略(默认是OnCreateAndWrite)：

- `StateTtlConfig.UpdateType.OnCreateAndWrite` - 仅在创建和写入时更新
- `StateTtlConfig.UpdateType.OnReadAndWrite` - 读取时也更新

数据在过期但还未被清理时的可见性配置如下(默认为NeverReturnExpired)：

- `StateTtlConfig.StateVisibility.NeverReturnExpired` - 不返回过期数据
- `StateTtlConfig.StateVisibility.ReturnExpiredIfNotCleanedUp` - 会返回过期但未清理的数据

`NeverReturnExpired` 情况下，过期数据就像不存在一样，不管是否被物理删除。这对于不能访问过期数据的场景下非常有用，比如敏感数据。 `ReturnExpiredIfNotCleanedUp` 在数据被物理删除前都会返回。

**注意：**

- 状态上次的修改时间会和数据一起保存在state backend中，因此开启该特性会增加状态数据的存储。Heap State backend会额外存储一个包括用户状态以及时间戳的 Java 对象，RocksDB state backend会在每个状态值(list或map的每个元素)序列化后增加 8 个字节。
- 暂时只支持基于 *processing time* 的 TTL。
- 尝试从 checkpoint/savepoint 进行恢复时，TTL 的状态（是否开启）必须和之前保持一致，否则会遇到 “StateMigrationException”。
- TTL 的配置并不会保存在 checkpoint/savepoint 中，仅对当前 Job 有效。
- 当前开启 TTL 的 map state 仅在用户值序列化器支持 null 的情况下，才支持用户值为 null。如果用户值序列化器不支持 null， 可以用 `NullableSerializer` 包装一层。

### 4.1 过期数据的清理

默认情况下，过期数据会在读取的时候被删除，例如 `ValueState#value`，同时会有后台线程定期清理（如果 StateBackend 支持的话）。可以通过 `StateTtlConfig` 配置关闭后台清理：

```java
import org.apache.flink.api.common.state.StateTtlConfig;

StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    .disableCleanupInBackground()
    .build();
```

可以对过期数据清理配置更细粒度的清理策略，当前的实现中HeapStateBackend依赖增量数据清理，RocksDBStateBackend利用压缩过滤器进行后台清理。

### 4.2 全量快照进行清理

另外，你可以启用全量快照时进行清理的策略，这可以减少整个快照的大小。当前实现中不会清理本地的状态，但从上次快照恢复时，不会恢复那些已经删除的过期数据。 该策略可以通过 `StateTtlConfig` 配置进行配置：

