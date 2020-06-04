# Reducer

## 1. Reducer核心思想

Reducer有3个主要阶段：

1. Shuffle：Reducer是Mapper的分组输出，在阶段框架中，每一个Reducer通过HTTP获取所有Mapper相关分区的输出，即多个Mapper分区输出到一个Reducer中，默认为Hash映射

2. Sort：MapReduce框架按照Key将Reducer的输入分组，因为在这个阶段，不同的Mapper可能输出相同的Key。Shuffle阶段与Sort阶段同时发生，例如：Reducer在获取Mapper输出时已经将它们合并

3. SecondarySort：如果在对中间结果进行分组时需要改变Key的分组/排序规则，可以通过自定义Comparator，

	JobConf.setOutputValueGroupingComparator,JobConf.setOutputKeyComparatorClass可用于控制如何对中间记录的key进行分组，这些键key结合使用以模拟对值的二级排序。

## 2. 源码

```java
/** 
 * Reduces a set of intermediate values which share a key to a smaller set of
 * values.
 *
 * 将一组共享key的中间数据值reduce到一组
 * 
 * <p>The number of <code>Reducer</code>s for the job is set by the user via 
 * {@link JobConf#setNumReduceTasks(int)}. <code>Reducer</code> implementations 
 * can access the {@link JobConf} for the job via the 
 * {@link JobConfigurable#configure(JobConf)} method and initialize themselves. 
 * Similarly they can use the {@link Closeable#close()} method for
 * de-initialization.</p>
 *
 * 作业的Reducer数量可以通过JobConf.setNumReduceTasks由用户自定义设置
 *
 * <p><code>Reducer</code> has 3 primary phases:</p>
 *
 * <P>Reducer有3个主要阶段：</P>
 * 1. Shuffle：Reducer是Mapper的分组输出，在阶段框架中，每一个Reducer通过HTTP获取所有Mapper相关分区的输出，即多个Mapper分区输出到一个Reducer中，默认为Hash映射
 * 2. Sort：MapReduce框架按照Key将Reducer的输入分组，因为在这个阶段，不同的Mapper可能输出相同的Key
 * Shuffle阶段与Sort阶段同时发生，例如：Reducer在获取Mapper输出时已经将它们合并
 * 3. SecondarySort：如果在对中间结果进行分组时需要改变Key的分组/排序规则，可以通过自定义Comparator，
 * JobConf.setOutputValueGroupingComparator,JobConf.setOutputKeyComparatorClass可用于控制如何对中间记录的key进行分组，这些键key结合使用以模拟对值的二级排序
 * <ol>
 *   <li>
 *   
 *   <b id="Shuffle">Shuffle</b>
 *   
 *   <p><code>Reducer</code> is input the grouped output of a {@link Mapper}.
 *   In the phase the framework, for each <code>Reducer</code>, fetches the 
 *   relevant partition of the output of all the <code>Mapper</code>s, via HTTP. 
 *   </p>
 *   </li>
 *   
 *   <li>
 *   <b id="Sort">Sort</b>
 *
 *   reducer
 *   <p>The framework groups <code>Reducer</code> inputs by <code>key</code>s 
 *   (since different <code>Mapper</code>s may have output the same key) in this
 *   stage.</p>
 *   
 *   <p>The shuffle and sort phases occur simultaneously i.e. while outputs are
 *   being fetched they are merged.</p>
 *
 *   <p>shuffle和sort阶段同时发生，即在获取输出时将它们合并</p>
 *
 *   <b id="SecondarySort">SecondarySort</b>
 *
 *   <b>二次排序</b>
 *
 *   <p>If equivalence rules for keys while grouping the intermediates are 
 *   different from those for grouping keys before reduction, then one may 
 *   specify a <code>Comparator</code> via 
 *   {@link JobConf#setOutputValueGroupingComparator(Class)}.Since 
 *   {@link JobConf#setOutputKeyComparatorClass(Class)} can be used to 
 *   control how intermediate keys are grouped, these can be used in conjunction 
 *   to simulate <i>secondary sort on values</i>.</p>
 *   
 *   
 *   For example, say that you want to find duplicate web pages and tag them 
 *   all with the url of the "best" known example. You would set up the job 
 *   like:
 *   <ul>
 *     <li>Map Input Key: url</li>
 *     <li>Map Input Value: document</li>
 *     <li>Map Output Key: document checksum, url pagerank</li>
 *     <li>Map Output Value: url</li>
 *     <li>Partitioner: by checksum</li>
 *     <li>OutputKeyComparator: by checksum and then decreasing pagerank</li>
 *     <li>OutputValueGroupingComparator: by checksum</li>
 *   </ul>
 *   </li>
 *   
 *   <li>   
 *   <b id="Reduce">Reduce</b>
 *   
 *   <p>In this phase the 
 *   {@link #reduce(Object, Iterator, OutputCollector, Reporter)}
 *   method is called for each <code>&lt;key, (list of values)&gt;</code> pair in
 *   the grouped inputs.</p>
 *   <p>The output of the reduce task is typically written to the 
 *   {@link FileSystem} via 
 *   {@link OutputCollector#collect(Object, Object)}.</p>
 *   </li>
 * </ol>
 * 
 * <p>The output of the <code>Reducer</code> is <b>not re-sorted</b>.</p>
 * 
 * <p>Example:</p>
 * <p><blockquote><pre>
 *     public class MyReducer&lt;K extends WritableComparable, V extends Writable&gt; 
 *     extends MapReduceBase implements Reducer&lt;K, V, K, V&gt; {
 *     
 *       static enum MyCounters { NUM_RECORDS }
 *        
 *       private String reduceTaskId;
 *       private int noKeys = 0;
 *       
 *       public void configure(JobConf job) {
 *         reduceTaskId = job.get(JobContext.TASK_ATTEMPT_ID);
 *       }
 *       
 *       public void reduce(K key, Iterator&lt;V&gt; values,
 *                          OutputCollector&lt;K, V&gt; output, 
 *                          Reporter reporter)
 *       throws IOException {
 *       
 *         // Process
 *         int noValues = 0;
 *         while (values.hasNext()) {
 *           V value = values.next();
 *           
 *           // Increment the no. of values for this key
 *           ++noValues;
 *           
 *           // Process the &lt;key, value&gt; pair (assume this takes a while)
 *           // ...
 *           // ...
 *           
 *           // Let the framework know that we are alive, and kicking!
 *           if ((noValues%10) == 0) {
 *             reporter.progress();
 *           }
 *         
 *           // Process some more
 *           // ...
 *           // ...
 *           
 *           // Output the &lt;key, value&gt; 
 *           output.collect(key, value);
 *         }
 *         
 *         // Increment the no. of &lt;key, list of values&gt; pairs processed
 *         ++noKeys;
 *         
 *         // Increment counters
 *         reporter.incrCounter(NUM_RECORDS, 1);
 *         
 *         // Every 100 keys update application-level status
 *         if ((noKeys%100) == 0) {
 *           reporter.setStatus(reduceTaskId + " processed " + noKeys);
 *         }
 *       }
 *     }
 * </pre></blockquote>
 * 
 * @see Mapper
 * @see Partitioner
 * @see Reporter
 * @see MapReduceBase
 */
@InterfaceAudience.Public
@InterfaceStability.Stable
public interface Reducer<K2, V2, K3, V3> extends JobConfigurable, Closeable {
  
  /** 
   * <i>Reduces</i> values for a given key.  
   * 
   * <p>The framework calls this method for each 
   * <code>&lt;key, (list of values)&gt;</code> pair in the grouped inputs.
   * Output values must be of the same type as input values.  Input keys must 
   * not be altered. The framework will <b>reuse</b> the key and value objects
   * that are passed into the reduce, therefore the application should clone
   * the objects they want to keep a copy of. In many cases, all values are 
   * combined into zero or one value.
   * </p>
   *
   * MR框架对每一个(Key, list(value))调用reduce方法，多个输入值可以合并为0个或1个输出值
   *   
   * <p>Output pairs are collected with calls to  
   * {@link OutputCollector#collect(Object,Object)}.</p>
   *
   * <p>Applications can use the {@link Reporter} provided to report progress 
   * or just indicate that they are alive. In scenarios where the application 
   * takes a significant amount of time to process individual key/value 
   * pairs, this is crucial since the framework might assume that the task has 
   * timed-out and kill that task. The other way of avoiding this is to set 
   * <a href="{@docRoot}/../hadoop-mapreduce-client/hadoop-mapreduce-client-core/mapred-default.xml#mapreduce.task.timeout">
   * mapreduce.task.timeout</a> to a high-enough value (or even zero for no 
   * time-outs).</p>
   * 
   * @param key the key.
   * @param values the list of values to reduce.
   * @param output to collect keys and combined values.
   * @param reporter facility to report progress.
   */
  void reduce(K2 key, Iterator<V2> values,
              OutputCollector<K3, V3> output, Reporter reporter)
    throws IOException;

}
```

