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

你必须创建一个 `StateDescriptor`，才能得到对应的状态句柄。 这保存了状态名称（正如我们稍后将看到的，你可以创建多个状态，并且它们必须具有唯一的名称以便可以引用它们）， 状态所持有值的类型，并且可能包含用户指定的函数，例如`ReduceFunction`。 根据不同的状态类型，可以创建`ValueStateDescriptor`，`ListStateDescriptor`， `ReducingStateDescriptor`，`FoldingStateDescriptor` 或 `MapStateDescriptor`。

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