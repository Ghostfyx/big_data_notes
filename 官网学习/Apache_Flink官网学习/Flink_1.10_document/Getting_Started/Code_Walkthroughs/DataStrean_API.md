# DataStream API

Apache Flink提供了用于构建鲁棒性强，有状态流应用程序的API。提供了对时间和状态的细粒度控制，从而实现高级事件驱动系统。本指南中，将学习如果使用Flink的DataStream API构建有状态的流式应用程序。

## 1. What Are You Building?

在数字时代，信用卡欺诈已成为人们日益关注的问题。 犯罪分子通过进行诈骗或侵入不安全的系统来窃取信用卡号码。 偷窃号码的测试是通过一次或多次小额购买(通常是1美元或更少)进行的。 如果测试通过，那么将进行更大金额的购买，以获得可以出售或自己保留的物品。

在本教程中，将构建一个欺诈检测系统，以提醒可疑的信用卡交易。 使用一组简单的规则，将了解Flink如何使实现复杂业务逻辑并实时采取行动。

## 2. Prerequisites

本演练假定对Java或Scala有所了解，但是即使来自其他编程语言，也应该能够继续学习。

## 3. Get Help

如果遇到困难，请查看[community support resources](https://flink.apache.org/gettinghelp.html)。 特别是，Apache Flink的[user mailing list](https://flink.apache.org/community.html#mailing-lists)一直被评为所有Apache项目中最活跃的邮件列表之一，也是快速获得帮助的好方法。

## 4. How to Follow Along

如果要继续学习，则需要一台具有以下功能的计算机：

- JDK 8或JDK 11
- Maven

提供的Flink Maven Archetype将快速创建具有所有必要依赖项的框架项目，因此您只需要专注于填写业务逻辑即可。 这些依赖项包括flink-streaming-java(它是所有Flink流媒体应用程序的核心依赖项)和flink-walkthrough-common(具有数据生成器和其他特定于此演练的类)的依赖项。

PS：FraudDetectionJob相关代码参考[flink学习示例代码](https://github.com/Ghostfyx/flink_learning)

## 5. Breaking Down the Code

逐步介绍这两个文件的代码。 FraudDetectionJob类定义应用程序的数据流，而FraudDetector类定义检测欺诈性交易的功能的业务逻辑。

开始描述如何在FraudDetectionJob类的main方法中组装Job。

### The Execution Environment

第一行设置StreamExecutionEnvironment。 StreamExecutionEnvironment用于为Job设置属性，创建源以及最终触发Job执行。

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getStreamExecutionEnvironment()
```

### Creating a Source

source将来源于外部系统(例如：Apache Kafka, Rabbit MQ, or Apache Pulsar)的数据接入到Flink Jobs中。

```java
DataStream<Transaction> transactions = env
    .addSource(new TransactionSource())
    .name("transactions");
```

### Partitioning Events & Detecting Fraud

交易流包含来自大量用户的大量交易，因此需要并行处理多个欺诈检测任务。 由于欺诈是按帐户进行的，因此必须确保欺诈帐户操作的同一并行任务处理同一帐户的所有交易。

为确保同一物理任务处理特定键的所有记录，可以使用DataStream.keyBy对流进行分区。process()方法添加了一个操作，该操作将函数应用于数据流各个分区中的元素。通常情况下，process操作在keyby之后执行，在这种情况下，FraudDetector与keyby在同一上下文执行。

```java
DataStream<Alert> alerts = transactions
    .keyBy(Transaction::getAccountId)
    .process(new FraudDetector())
    .name("fraud-detector");
```

### Outputting Results

sink将DataStream写入外部系统； 例如Apache Kafka，Cassandra和AWS Kinesis。 AlertSink使用日志级别INFO记录每个Alert记录，而不是将其写入持久性存储，主要是为了测试期间方便查询结果。

```java
alerts.addSink(new AlertSink());
```

### Executing the Job

Flink应用程序是延迟构建的，并仅在完全构建后才交付给集群以执行。 调用StreamExecutionEnvironment＃execute开始执行Job并为其命名。

```java
env.execute("Fraud Detection");
```

### The Fraud Detector

欺诈检测器实现KeyedProcessFunction抽象类。 每个交易事件都会调用其方法KeyedProcessFunction＃processElement。 第一个版本会在每笔交易时发出警报并打印日志。

后续步骤将使用更有意义的业务逻辑扩展的欺诈检测器。

## 6. Writing a Real Application (v1)

对于第一个版本，欺诈检测器应针对任何进行小额交易然后立即进行大笔交易的帐户输出警报。 小的小于$ 1.00，大的大于$ 500。 想象一下，您的欺诈检测器为特定帐户处理以下交易流。

![](https://ci.apache.org/projects/flink/flink-docs-release-1.10/fig/fraud-transactions.svg)

交易3和4应该标记为欺诈，因为这是一笔小交易，0.09美元，然后是一笔大交易，510美元。另外，交易7、8和9不是欺诈，因为少量的$ 0.02不会立即跟随大笔的交易；而是有一个中间事务破坏了模式。

为此，欺诈检测者必须记住事件之间的信息。大型交易只有在前一个交易规模较小的情况下才具有欺诈性。记住事件之间的信息需要状态，这就是为什么我们决定使用KeyedProcessFunction的原因。它提供了对状态和时间的细粒度控制，这将使我们能够在整个演练中发展出具有更复杂要求的算法。

最直接的实现是布尔标志，该标志在处理小事务时都会设置。当进行大笔交易时，您可以简单地检查是否为该帐户设置了标志。

但是，仅将标志实现为FraudDetector类中的成员变量是行不通的。 Flink使用相同的FraudDetector对象实例处理多个帐户的交易，这意味着如果帐户A和B通过相同的FraudDetector实例进行路由，则帐户A的交易可以将标志设置为true，然后帐户B的交易可以引起虚假警报。我们当然可以使用诸如Map之类的数据结构来跟踪各个键的标志，但是，简单的成员变量将不会容错，并且在发生故障时会丢失其所有信息。因此，如果应用程序必须重新启动以从故障中恢复，欺诈检测器可能会丢失警报。

为了解决这些挑战，Flink提供了用于容错的原始状态，这些原始状态几乎与常规成员变量一样易于使用。

Flink中最基本的状态类型是ValueState，这是一种向其包装的变量添加容错功能的数据类型。 ValueState是键状态的一种形式，这意味着它仅适用于在键上下文中应用的操作。 紧随DataStream＃keyBy之后的任何操作。  操作的键控状态会自动限制在当前正在处理的记录的键范围内。 在此示例中，键是当前交易的帐户ID(由keyBy()声明)，并且FraudDetector维护每个帐户的独立状态。 使用ValueStateDescriptor创建ValueState，其中包含有关Flink应如何管理变量的元数据。 功能开始处理数据之前，应先注册状态。 正确的选择是open()方法。

```java
public class FraudDetector extends KeyedProcessFunction<Long, Transaction, Alert> {

    private static final long serialVersionUID = 1L;

    private transient ValueState<Boolean> flagState;

    @Override
    public void open(Configuration parameters) {
        ValueStateDescriptor<Boolean> flagDescriptor = new ValueStateDescriptor<>(
                "flag",
                Types.BOOLEAN);
        flagState = getRuntimeContext().getState(flagDescriptor);
    }
```

ValueState是包装器类，类似于Java标准库中的AtomicReference或AtomicLong。 它提供了三种与其内容进行交互的方法； 更新设置状态，值获取当前值，清除清除其内容。 如果特定键的状态为空，例如在应用程序开始时或在调用ValueState＃clear之后，则ValueState＃value将返回null。 不能保证系统可以识别对ValueState＃value返回的对象的修改，因此必须使用ValueState＃update进行所有更改。此外，容错功能由Flink在后台自动管理，因此可以像使用任何标准变量一样与之交互。

```java
  @Override
    public void processElement(
            Transaction transaction,
            Context context,
            Collector<Alert> collector) throws Exception {

        // Get the current state for the current key
        Boolean lastTransactionWasSmall = flagState.value();

        // Check if the flag is set
        if (lastTransactionWasSmall != null) {
            if (transaction.getAmount() > LARGE_AMOUNT) {
                // Output an alert downstream
                Alert alert = new Alert();
                alert.setId(transaction.getAccountId());

                collector.collect(alert);            
            }

            // Clean up our state
            flagState.clear();
        }

        if (transaction.getAmount() < SMALL_AMOUNT) {
            // Set the flag to true
            flagState.update(true);
        }
    }
```

对于每笔交易，欺诈检测器都会检查该帐户的标志状态。 请记住，ValueState始终限于当前键，即帐户。 如果该标志不为空，则该帐户看到的最后一笔交易很小，因此，如果该笔交易的金额很大，那么检测器将输出欺诈警报。

在检查之后，将无条件清除标志状态。 当前事务导致欺诈警报，并且模式已结束，或者当前事务未引起警报，并且模式已中断，需要重新启动。

最后，检查交易金额以查看交易金额是否较小。 如果是这样，那么将设置该标志，以便可以在下一个事件中对其进行检查。 请注意，`ValueState <Boolean>`实际上具有三个状态，即unset(null)，true和false，因为所有ValueState都是可为空的。 该作业仅使用unset(null)和true来检查是否设置了该标志。

## 7. Fraud Detector v2: State + Time = ❤️

诈骗者不会为了减少发现测试交易的机会，等很久才进行大量购买。  例如，假设您要为欺诈检测器设置1分钟超时； 即在之前的示例中，如果交易3和4发生在彼此之间1分钟之内，则交易3和4仅会被视为欺诈。 Flink的KeyedProcessFunction允许设置计时器，该计时器在将来的某个时间点调用回调方法。

如何修改作业以满足新的要求：

- 每当标志设置为true时，还要设置1分钟的计时器
- 当计时器触发时，通过清除其状态来重置标志
- 如果清除了该标志，则应取消计时器

要取消计时器，必须记住设置的时间，并记住隐含的状态，因此首先要创建一个计时器状态以及标志状态。

```java
private transient ValueState<Boolean> flagState;
private transient ValueState<Long> timerState;

@Override
public void open(Configuration parameters) {
    ValueStateDescriptor<Boolean> flagDescriptor = new ValueStateDescriptor<>(
        "flag",
        Types.BOOLEAN);
    flagState = getRuntimeContext().getState(flagDescriptor);
    ValueStateDescriptor<Long> timerDescriptor = new ValueStateDescriptor<>(
        "timer-state",
        Types.LONG);
    timerState = getRuntimeContext().getState(timerDescriptor);
}
```

使用包含计时器服务的上下文调用KeyedProcessFunction＃processElement。计时器服务可用于查询当前时间，注册计时器和删除计时器。 这样，可以在每次设置标志时将计时器设置为将来1分钟，并将时间戳存储在timerState中。

```java
if (transaction.getAmount() < SMALL_AMOUNT) {
    // set the flag to true
    flagState.update(true);

    // set the timer and timer state
    long timer = context.timerService().currentProcessingTime() + ONE_MINUTE;
    context.timerService().registerProcessingTimeTimer(timer);
    timerState.update(timer);
}
```

处理时间是系统时间，由运行操作机器的系统时钟确定。计时器触发时，它将调用KeyedProcessFunction＃onTimer。 重写此方法实现回调以重置标志。

```java
@Override
public void onTimer(long timestamp, OnTimerContext ctx, Collector<Alert> out) {
    // remove flag after 1 minute
    timerState.clear();
    flagState.clear();
}
```

最后，取消计时器，需要删除已注册的计时器并删除计时器状态。 可以将其包装在辅助方法中，然后调用此方法，而不是flagState.clear()。

```java
private void cleanUp(Context ctx) throws Exception {
    // delete timer
    Long timer = timerState.value();
    ctx.timerService().deleteProcessingTimeTimer(timer);

    // clean up all state
    timerState.clear();
    flagState.clear();
}
```

