# Java Lambda Expressions

Java 8引入了一些新的语言功能，旨在更快，更清晰地编码。 它具有最重要的功能，即所谓的“ Lambda表达式”，为函数式编程打开了大门。 Lambda表达式允许以直接方式实现和传递函数，而无需声明其他（匿名）类。

link支持对Java API的所有运算符使用lambda表达式，但是，每当lambda表达式使用Java泛型时，您都需要显式声明类型信息。

## 1. 例子和限制

下面例子说明了如何在Flink Java API中实现一个简单的例子，map()函数使用了一个Lambda表达式，由于Java编译器可以推断map()函数的输入和输出参数的类型，因此无需声明。

```java
env.fromElements(1, 2, 3)
// returns the squared i
.map(i -> i*i)
.print();
```

Flink可以从方法签名OUT的实现中自动提取结果类型信息，例如：`map(IN value)`。但是，有些函数例如：`void flatMap(IN value，Collector <OUT> out)`的已被Java编译器编译为`void flatMap(IN value，Collector out)`。 这使得Flink无法自动推断输出类型的类型信息。

对于上述情况，Flink会抛出一个类似下面的异常：

```
org.apache.flink.api.common.functions.InvalidTypesException: The generic type parameters of 'Collector' are missing.
    In many cases lambda methods don't provide enough information for automatic type extraction when Java generics are involved.
    An easy workaround is to use an (anonymous) class instead that implements the 'org.apache.flink.api.common.functions.FlatMapFunction' interface.
    Otherwise the type has to be specified explicitly using type information.
```

这种情况下，可以有多种方式解决：

- 类型信息需要被额外指定。

```java
import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.api.java.DataSet;
import org.apache.flink.util.Collector;

DataSet<Integer> input = env.fromElements(1, 2, 3);

// collector type must be declared
input.flatMap((Integer number, Collector<String> out) -> {
    StringBuilder builder = new StringBuilder();
    for(int i = 0; i < number; i++) {
        builder.append("a");
        out.collect(builder.toString());
    }
})
// provide type information explicitly
.returns(Types.STRING)
// prints "a", "a", "aa", "a", "aa", "aaa"
.print();
```

- 使用一个公共类方法替代

```java
// use a class instead
env.fromElements(1, 2, 3)
    .map(new MyTuple2Mapper())
    .print();

public static class MyTuple2Mapper extends MapFunction<Integer, Tuple2<Integer, Integer>> {
    @Override
    public Tuple2<Integer, Integer> map(Integer i) {
        return Tuple2.of(i, i);
    }
}
```

- 使用匿名内部类

```java
// use an anonymous class instead
env.fromElements(1, 2, 3)
    .map(new MapFunction<Integer, Tuple2<Integer, Integer>> {
        @Override
        public Tuple2<Integer, Integer> map(Integer i) {
            return Tuple2.of(i, i);
        }
    })
    .print();
```

- 使用自定义类

```java
// or in this example use a tuple subclass instead
env.fromElements(1, 2, 3)
    .map(i -> new DoubleTuple(i, i))
    .print();

public static class DoubleTuple extends Tuple2<Integer, Integer> {
    public DoubleTuple(int f0, int f1) {
        this.f0 = f0;
        this.f1 = f1;
    }
}
```

