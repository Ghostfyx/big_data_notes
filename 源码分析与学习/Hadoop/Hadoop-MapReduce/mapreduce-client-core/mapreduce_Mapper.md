# Mapper

```java
/** 
 * Maps input key/value pairs to a set of intermediate key/value pairs.
 *
 * 将输入的键值对映射到中间值的键值对
 * 
 * <p>Maps are the individual tasks which transform input records into a 
 * intermediate records. The transformed intermediate records need not be of 
 * the same type as the input records. A given input pair may map to zero or 
 * many output pairs.</p>
 *
 * mapper是将输入记录转换为中间记录的单个任务。被转换的中间记录并不需要与输入记录类型相同。一个给定的输入可以有0个到多个输出
 * 
 * <p>The Hadoop Map-Reduce framework spawns one map task for each 
 * {@link InputSplit} generated by the {@link InputFormat} for the job.
 * <code>Mapper</code> implementations can access the {@link Configuration} for 
 * the job via the {@link JobContext#getConfiguration()}.
 *
 * Hadoop Map-Reduce计算框架为作业的InputFormat生成的每个InputSplit生成一个map任务。
 * mapper的实现类可以通过JobContext的getConfiguration方法访问Configuration
 *
 * <p>The framework first calls 
 * {@link #setup(org.apache.hadoop.mapreduce.Mapper.Context)}, followed by
 * {@link #map(Object, Object, org.apache.hadoop.mapreduce.Mapper.Context)}
 * for each key/value pair in the <code>InputSplit</code>. Finally 
 * {@link #cleanup(org.apache.hadoop.mapreduce.Mapper.Context)} is called.</p>
 *
 * mapper计算框架首先对InputSplit的每个(key, value)键值对记录调用 setup方法和map犯法
 *
 * <p>All intermediate values associated with a given output key are 
 * subsequently grouped by the framework, and passed to a {@link Reducer} to  
 * determine the final output. Users can control the sorting and grouping by 
 * specifying two key {@link RawComparator} classes.</p>
 *
 * 给定输出key关联的所有中间值都由框架进行分组，并传递给reducer确定最终输出，
 * 用户可以通过指定两个key的RawComparator类控制分组和排序
 *
 *
 * <p>The <code>Mapper</code> outputs are partitioned per 
 * <code>Reducer</code>. Users can control which keys (and hence records) go to 
 * which <code>Reducer</code> by implementing a custom {@link Partitioner}.
 *
 * Mapper的输出是每个Reducer的一部分输入，用户可以通过实现自定义实现Partitioner，计算keys的分区，从而控制发送到哪个Reducer
 * 
 * <p>Users can optionally specify a <code>combiner</code>, via 
 * {@link Job#setCombinerClass(Class)}, to perform local aggregation of the 
 * intermediate outputs, which helps to cut down the amount of data transferred 
 * from the <code>Mapper</code> to the <code>Reducer</code>.
 *
 * 用户可以通过job.setCombinerClass选择指定一个combiner，用于对中间输出结果的本地聚合，这有助于减少从Mapper端传输到Reducer端的传输数据量
 * 
 * <p>Applications can specify if and how the intermediate
 * outputs are to be compressed and which {@link CompressionCodec}s are to be
 * used via the <code>Configuration</code>.</p>
 *
 * 应用程序可以通过来指定是否压缩中间输出以及如何压缩中间输出(通过Configuration制定使用哪个CompressionCodec)
 *  
 * <p>If the job has zero
 * reduces then the output of the <code>Mapper</code> is directly written
 * to the {@link OutputFormat} without sorting by keys.</p>
 *
 * 如果当前Job没有reduce任务，那么Mapper的输出结果可以不需要根据key排序，直接被写入到OutputFormat
 * 
 * <p>Example:</p>
 * <p><blockquote><pre>
 * public class TokenCounterMapper 
 *     extends Mapper&lt;Object, Text, Text, IntWritable&gt;{
 *    
 *   private final static IntWritable one = new IntWritable(1);
 *   private Text word = new Text();
 *   
 *   public void map(Object key, Text value, Context context) throws IOException, InterruptedException {
 *     StringTokenizer itr = new StringTokenizer(value.toString());
 *     while (itr.hasMoreTokens()) {
 *       word.set(itr.nextToken());
 *       context.write(word, one);
 *     }
 *   }
 * }
 * </pre></blockquote>
 *
 * <p>Applications may override the
 * {@link #run(org.apache.hadoop.mapreduce.Mapper.Context)} method to exert
 * greater control on map processing e.g. multi-threaded <code>Mapper</code>s 
 * etc.</p>
 *
 * 应用可以重写run方法对map任务更好的控制，例如多线程
 * 
 * @see InputFormat
 * @see JobContext
 * @see Partitioner  
 * @see Reducer
 */
@InterfaceAudience.Public
@InterfaceStability.Stable
public class Mapper<KEYIN, VALUEIN, KEYOUT, VALUEOUT> {

  /**
   * The <code>Context</code> passed on to the {@link Mapper} implementations.
   */
  public abstract class Context
    implements MapContext<KEYIN,VALUEIN,KEYOUT,VALUEOUT> {
  }
  
  /**
   * Called once at the beginning of the task.
   *
   * 在map任务启动时会立即调用
   */
  protected void setup(Context context
                       ) throws IOException, InterruptedException {
    // NOTHING
  }

  /**
   * Called once for each key/value pair in the input split. Most applications
   * should override this, but the default is the identity function.
   *
   * 对input split中的每条键值对记录都会调用一次map方法，我们通过重现map方法实现业务逻辑
   */
  @SuppressWarnings("unchecked")
  protected void map(KEYIN key, VALUEIN value, 
                     Context context) throws IOException, InterruptedException {
    context.write((KEYOUT) key, (VALUEOUT) value);
  }

  /**
   * Called once at the end of the task.
   */
  protected void cleanup(Context context
                         ) throws IOException, InterruptedException {
    // NOTHING
  }
  
  /**
   * Expert users can override this method for more complete control over the
   * execution of the Mapper.
   * @param context
   * @throws IOException
   */
  public void run(Context context) throws IOException, InterruptedException {
    setup(context);
    try {
      while (context.nextKeyValue()) {
        map(context.getCurrentKey(), context.getCurrentValue(), context);
      }
    } finally {
      cleanup(context);
    }
  }
}
```

