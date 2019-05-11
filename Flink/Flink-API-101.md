- [Overview](#overview)
- [Set Up Configurations](#set-up-configurations)
  - [Set up the execution environment](#set-up-the-execution-environment)
  - [Set up the Execution Parallelism](#set-up-the-execution-parallelism)
  - [Set Up Time Characteristics](#set-up-time-characteristics)
  - [Set Up Timestamp and Watermark Generator](#set-up-timestamp-and-watermark-generator)
    - [Source Function](#source-function)
    - [User-defined Assigner](#user-defined-assigner)
- [Function Class](#function-class)
  - [Serializable](#serializable)
  - [Rich Functions](#rich-functions)
  - [ProcessFunction](#processfunction)
- [DataStream Transformation API](#datastream-transformation-api)
  - [Compare with Batch Processing](#compare-with-batch-processing)
  - [Basic transformations](#basic-transformations)
  - [KeyedStream Transformations](#keyedstream-transformations)
    - [DataStream.keyBy](#datastreamkeyby)
  - [Multistream transformations](#multistream-transformations)
    - [Select](#select)
    - [ConnectedStream](#connectedstream)
  - [Distribution(partition) transformations](#distributionpartition-transformations)
- [ProcessFunction in Detail](#processfunction-in-detail)
  - [Side Output](#side-output)
- [Stateful Operators](#stateful-operators)

# Overview
Flink supports all common data types that are available in Java and Scala. The most widely used types can be grouped into the following categories: Primitives, Java and Scala tuples, Scala case classes, POJOs, some special types such as Arrays, Lists, Maps, Enums. To structure a typical Flink streaming application:
1. Set up configurations
   Set up the execution environment, parallelism and time characteristics.
2. Read one or more streams from data sources:  
   ```StreamExecutionEnvironment.addSource()```
3. Apply streaming transformations to implement the application logic
4. Optionally output the result to one or more data sinks
5. Execute the program  
   Flink programs are executed **lazily**. Only when ```StreamExecutionEnvironment.execute()``` is called does the system trigger the execution of the program. The constructed plan is translated into a JobGraph and submitted to a JobManager for execution

# Set Up Configurations
## Set up the execution environment
We can use ```StreamExecutionEnvironment.getExecutionEnvironment``` to retrieve the execution environment. 
The result is a **remote** execution environment if the method is invoked from a submission client with a connection to a remote cluster. Otherwise, it returns a **local** environment. 

We can also explicitly create local or remote execution environments as follow:
```scala
// create a local stream execution environment 
val localEnv: StreamExecutionEnvironment.createLocalEnvironment()

// create a remote stream execution environment 
val remoteEnv = StreamExecutionEnvironment.createRemoteEnvironment(
    "host", // hostname of JobManager 
    1234, // port of JobManager process 
    "path/to/jarFile.jar") // JAR file to ship to the JobManager
```

## Set up the Execution Parallelism
We can:
* Use the Default parallelism  
By default, the parallelism of all operators is set as the parallelism of the application’s execution environment. If the application runs in a **local** execution environment the parallelism is set to match **the number of CPU cores**; if in a **Flink cluster**, the environment parallelism is set to the **default parallelism of the cluster** unless it is explicitly specified via the submission client.

* Override the parallelism of the environment  
You can also override the default parallelism of the environment, but you will no longer be able to control the parallelism of your application via the submission client:
    ```scala
    val env: StreamExecutionEnvironment.getExecutionEnvironment 
    env.setParallelism(32)
    ```
* Override the parallelism of specific Operator  
The default parallelism of an operator can be overridden by specifying it explicitly with ```setParallelism()``` method  
    ```scala
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    val defaultP = env.getParallelism 
    val result = env.addSource(new CustomSource)
        .map(new MyMapper).setParallelism(defaultP * 2)
        .print().setParallelism(2)
    ```

## Set Up Time Characteristics
We can use ```StreamExecutionEnvironment.setStreamTimeCharacteristic``` to choose ```TimeCharacteristic```:
* ```ProcessingTime```:local machine time when the event is being executed, low latency
* ```IngestionTime```: local machine time when the event enters stream operator, not recommended 
* ```EventTime```: using the timestamp from the event, or are assigned by the application at the sources, deterministic 

## Set Up Timestamp and Watermark Generator
Event time guarantees deterministic results, which is required by most applications. It is best practice to assign timestamps and generate watermarks as soon as possible, because most assigners make assumptions about the order of elements with respect to the timestamps when generating watermarks. 

### Source Function
// ToDo

### User-defined Assigner
The user-defined timestamp assigner is called on a stream and produce a new stream of timestamped elements and watermarks, of the the same DataStream type, e.g.:

```scala
val readings: DataStream[SensorReading] = env
  .addSource(new SensorSource) 
  .assignTimestampsAndWatermarks(new MyAssigner())
```

The ```MyAssigner``` can be either type ```AssignerWithPeriodicWatermarks``` or ```AssignerWithPunctuatedWatermarks```.

```AssignerWithPeriodicWatermarks``` instructs the system to emit watermarks and advance the event time **in fixed intervals of machine time**. Every n milliseconds, ```AssignerWithPeriodicWatermarks.getCurrentWatermark()```  is invoked. If the method returns a non-null value with a timestamp larger than the timestamp of the previous watermark, the new watermark is forwarded. 

E.g., the following assigner returns a watermark with the maximum timestamp minus a 1-minute tolerance interval

```scala
class PeriodicAssigner extends AssignerWithPeriodicWatermarks[SensorReading] {

  // 1 min in ms
  val bound: Long = 60 * 1000
  // the maximum observed timestamp
  var maxTs: Long = Long.MinValue

  override def getCurrentWatermark: Watermark = {
    new Watermark(maxTs - bound)
  }

  override def extractTimestamp(r: SensorReading, previousTS: Long): Long = {
    // update maximum timestamp
    maxTs = maxTs.max(r.timestamp)
    // return record timestamp
    r.timestamp
  }
}
```

```AssignerWithPeriodicWatermarks.getCurrentWatermark()``` generates watermarks when the input stream contains **special tuples or markers**. ```checkAndGetNextWatermark()``` is called for each event right after ```extractTimestamp()```. E.g.,

```scala
class PunctuatedAssigner extends AssignerWithPunctuatedWatermarks[SensorReading] {

  // 1 min in ms
  val bound: Long = 60 * 1000

  override def checkAndGetNextWatermark(r: SensorReading, extractedTS: Long): Watermark = {
    if (r.id == "sensor_1") {
      // emit watermark if reading is from sensor_1
      new Watermark(extractedTS - bound)
    } else {
      // do not emit a watermark
      null
    }
  }

  override def extractTimestamp(r: SensorReading, previousTS: Long): Long = {
    // assign record timestamp
    r.timestamp
  }
```

# Function Class
## Serializable
Flink serializes all function objects with **Java serialization** to ship them to the worker processes. **Everything contained in a user function must be serializable.**

If your function requires a non-serializable object instance, you can either implement it as a rich function and initialize the non-serializable field in the open() method or override the Java serialization and de-serialization methods.

## Rich Functions
When using a rich function, you can implement two additional methods to the function’s lifecycle:

 * ```open()``` method is an initialization method for the rich function. It is called once per task before a transformation method like filter or map is called. open() is typically used for setup work that needs to be done only once. 

* ```close()``` method is a finalization method for the function and it is called once per task after the last call of the transformation method. Thus, it is commonly used for cleanup and releasing resources.

open() and close() are called **once per task**. In the example below, open() and close() will be executed 4 times because the parallelism is 4.
```scala
val inputStream: DataStream[(Int, Int, Int)] = env.fromElements(
      (1, 2, 2), (2, 3, 1), (2, 2, 4))
val filteredSensors = inputStream.filter(new TestRichFilter()).setParallelism(4)

class TestRichFilter extends RichFilterFunction[(Int, Int, Int)] {
    override def open(parameters: Configuration): Unit = println("open test")

    override def filter(r: (Int, Int, Int)): Boolean = r._1 >= 2

    override def close(): Unit = println("close test")
  }
```

In RichFunction class, ```getRuntimeContext()``` method provides access to the function’s RuntimeContext such as the function’s parallelism, its subtask index, and the name of the task that executes the function, methods for accessing partitioned state.

## ProcessFunction
ProcessFunction are commonly used to implement **custom logic** for which predefined windows and transformations might not be suitable, which can: 
* Access record timestamps and watermarks and register timers that trigger at a specific time in the future
* Emit records to multiple output streams with side outputs

For detail please refer to the [ProcessFunction in Detail](#processfunction-in-detail) section.


# DataStream Transformation API
## Compare with Batch Processing
Batch processing and stream processing is different. Let's look at an example here.
```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setParallelism(1)

val inputStream: DataStream[(Int, Int)] = env.fromElements(
      (1, 2), (2, 3), (2, 2), (1, 5))

val resultStream: DataStream[(Int, Int)] = inputStream
      .keyBy(0) // key on first field of the tuple
      .sum(1)   // sum the second field of the tuple

resultStream.print()
``` 

In batch processing program like Spark, data are all collected then processed. That is, The first RDD consists of 4 records, keyBy() will generate 2 RDDs, and each RDD will execute sum() operation. So the log in the console will be:
``` 
(1, 7)
(2, 5)
```

In Flink, the job graph sets up pipeline, which will be triggered whenever a new record comes. In this code there are 4 records, so the job graph will be executed 4 times and the log is as follow:
```
(1,2)
(2,3)
(2,5)
(1,7)
```

## Basic transformations
Basic transformations are transformations on individual events: 
* DataStream.map() -> ```MapFunction``` interface
* DataStream.filter() -> ```FilterFunction``` interface
* DataStream.flatMap() -> ```FlatMapFunction``` interface 

## KeyedStream Transformations 
We can use ```DataStream.keyBy``` to convert a DataStream to a KeyedStream, which has the following operations: sum, min, minBy, max, maxBy, reduce, fold.

### DataStream.keyBy
In Flink, keys are **not predefined** in the input types. Instead, keys are **defined as functions over the input data**. We have three ways to define keys: ```field position```, ```field expressions```, ```KeySelector```. KeySelector functions is my preferred way because it is **strongly typed**.

**Field position**: If the data type is a tuple, keys can be defined by simply using the :
```scala
val input: DataStream[(Int, String, Long)] = ... 
val keyed = input.keyBy(1)
```

**Field expressions**: For tuples, POJOs, and case classes, and nested fields:
```scala
case class SensorReading( 
    id: String, 
    timestamp: Long, 
    temperature: Double)

val sensorStream: DataStream[SensorReading] = ... 
val keyedSensors = sensorStream.keyBy("id")
``` 

**KeySelector** function receives an input item and returns a key. The key does not necessarily have to be a field of the input event but can be **derived** using arbitrary computations.

```scala
val input : DataStream[(Int, Int)] = ...
val keyedStream = input.keyBy(value => math.max(value._1, value._2))
```

## Multistream transformations
The ```DataStream.union()``` method merges two or more DataStreams of the **same** type.  

### Select
Each incoming event can be routed to zero, one, or more output streams. Hence, split can also be used to **filter or replicate** events.

**DataStream.split()** method receives an **OutputSelector** that defines how stream elements are assigned to named outputs.

```scala
val inputStream: DataStream[(Int, String)] = ...
val splitted = inputStream.split(t => if (t._1 > 1000) Seq("large") else Seq("small"))

val large = splitted.select("large") 
val small = splitted.select("small") 
val all = splitted.select("small", "large")
```
The code above is an example.

### ConnectedStream  
The ```DataStream.connect()``` method merges two streams of (possibly) **different type** and returns a ```ConnectedStreams``` type. 

The doc has one example:
> An example for the use of connected streams would be to apply rules that change over time onto another stream. One of the connected streams has the rules, the other stream the elements to apply the rules to. The operation on the connected stream maintains the current set of rules in the state. It may receive either a rule update and update the state or a data element and apply the rules in the state to the element.

The ConnectedStreams object provides map() and flatMap() methods that expect a ```CoMapFunction``` and ```CoFlatMapFunction``` as argument respectively. Both functions are typed on the types of the first and second input stream and on the type of the output stream and define two methods for each input: 
```scala 
// IN1: the type of the first input stream 
// IN2: the type of the second input stream 
// OUT: the type of the output elements 
CoMapFunction[IN1, IN2, OUT]
    > map1(IN1): OUT
    > map2(IN2): OUT
```

By default, events of both streams are **randomly** assigned to operator instances. In order to achieve deterministic transformations on ConnectedStreams, connect() can be combined with ```keyBy()``` or ```broadcast()```.

E.g., in order for apply all the rules to each element, we may do:
```scala
val rulesDataConnect = element.connect(rules.broadcast())
```
In order to apply only the associated rules to each element, we may do:
```scala
val rulesDataConnect = element.keyBy(0).connect(rule.keyBy(0)
```  

## Distribution(partition) transformations 
Distribution transformations reorganize stream events.

When building applications with the DataStream API the system automatically chooses data partitioning strategies, or we can control the partitioning strategies at the application level manually.

* ```DataStream.shuffle```: events are distributed **randomly** according to a uniform distribution
* ```DataStream.rebalance```: events are evenly distributed to successor tasks in a **round-robin** fashion
* ```DataStream.rescale```: events are evenly distributed to successor tasks in a **round-robin** fashion only to a **subset** of successor tasks
* ```DataStream.Broadcast```: events are sent to **all** successor tasks
* ```DataStream.Global```: events are sent to the **first** parallel task of the successor
* ```DataStream.Custom```: define your partition strategy by using the ```partitionCustom()``` method with a ```Partitioner``` object, e.g., the following Partitioner send all negative numbers  to the first task and all other numbers to the second task
  ```scala 
    val numbers: DataStream[(Int)] = ... 
    numbers.partitionCustom(myPartitioner, 0)

    object myPartitioner extends Partitioner[Int] {
        override def partition(key: Int, numPartitions: Int): Int = {      
            if (key < 0) 0 else 1
        } 
    }
  ```
Finished

# ProcessFunction in Detail
Currently, Flink provides eight different process functions: ```ProcessFunction```, ```KeyedProcessFunction```, ```CoProcessFunction```, ```ProcessJoinFunction```, ```BroadcastProcessFunction```, ```KeyedBroadcastProcessFunction```, ```ProcessWindowFunction```, and ```ProcessAllWindowFunction```. These functions are applicable in different contexts. However, they have a very similar feature set.

They provides the following two methods:

* ```processElement(v: IN, ctx: Context, out: Collector[OUT])``` This method is  called for **each record** of the stream. Result records are emitted by passing them to the Collector. The Context object can emit records to side outputs and give access to the timestamp, the key of the current record, and to a ```TimerService```. 

  TimeService has the following methods: currentProcessingTime(), currentWatermark(), registerProcessingTimeTimer(timestamp: Long),registerEventTimeTimer(timestamp: Long), 
deleteProcessingTimeTimer(timestamp: Long),
deleteEventTimeTimer(timestamp: Long). They are all self-explained. 

* ```onTimer(timestamp: Long, ctx:OnTimerContext, out: Collector[OUT])``` is a callback function that is invoked when a previously registered timer triggers. The timestamp argument gives the timestamp of the firing timer and the Collector allows records to be emitted. The OnTimerContext provides the same services as the Context object of the processElement() method and also returns the time domain of the firing trigger. **For each key and timestamp, exactly one timer can be registered**, which means each key can have multiple timers but only one for each timestamp.

## Side Output
The ```split``` operator allows splitting a stream into multiple streams of the same type; Side outputs allows emitting multiple streams from a function with possibly **different types**. A side output is identified by an ```OutputTag[X]``` object, where X is the type of the resulting side output stream. Process functions can emit a record to one or more side outputs via the Context object.


 
# Stateful Operators
