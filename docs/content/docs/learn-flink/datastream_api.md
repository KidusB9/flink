---
title: Intro to the DataStream API
weight: 3
type: docs
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

# Intro to the DataStream API

The focus of this training is to broadly cover the DataStream API well enough that you will be able
to get started writing streaming applications.

## What can be Streamed?

Flink's DataStream APIs will let you stream anything they can serialize. Flink's
own serializer is used for

- basic types, i.e., String, Long, Integer, Boolean, Array
- composite types: Tuples, POJOs

and Flink falls back to Kryo for other types. It is also possible to use other serializers with
Flink. Avro, in particular, is well supported.

### Java tuples and POJOs

Flink's native serializer can operate efficiently on tuples and POJOs.

#### Tuples

For Java, Flink defines its own `Tuple0` thru `Tuple25` types.

```java
Tuple2<String, Integer> person = Tuple2.of("Fred", 35);

// zero based index!  
String name = person.f0;
Integer age = person.f1;
```

#### POJOs

Flink recognizes a data type as a POJO type (and allows “by-name” field referencing) if the following conditions are fulfilled:

- The class is public and standalone (no non-static inner class)
- The class has a public no-argument constructor
- All non-static, non-transient fields in the class (and all superclasses) are either public (and
  non-final) or have public getter- and setter- methods that follow the Java beans naming
  conventions for getters and setters.

Example:

```java
public class Person {
    public String name;  
    public Integer age;  
    public Person() {}
    public Person(String name, Integer age) {  
        . . .
    }
}  

Person person = new Person("Fred Flintstone", 35);
```

Flink's serializer [supports schema evolution for POJO types]({{< ref "docs/dev/datastream/fault-tolerance/serialization/schema_evolution" >}}#pojo-types).

## A Complete Example

This example takes a stream of records about people as input, and filters it to only include the adults.

```java
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.api.common.functions.FilterFunction;

public class Example {

    public static void main(String[] args) throws Exception {
        final StreamExecutionEnvironment env =
                StreamExecutionEnvironment.getExecutionEnvironment();

        DataStream<Person> flintstones = env.fromData(
                new Person("Fred", 35),
                new Person("Wilma", 35),
                new Person("Pebbles", 2));

        DataStream<Person> adults = flintstones.filter(new FilterFunction<Person>() {
            @Override
            public boolean filter(Person person) throws Exception {
                return person.age >= 18;
            }
        });

        adults.print();

        env.execute();
    }

    public static class Person {
        public String name;
        public Integer age;
        public Person() {}

        public Person(String name, Integer age) {
            this.name = name;
            this.age = age;
        }

        public String toString() {
            return this.name.toString() + ": age " + this.age.toString();
        }
    }
}
```

### Stream execution environment

Every Flink application needs an execution environment, `env` in this example. Streaming
applications need to use a `StreamExecutionEnvironment`.

The DataStream API calls made in your application build a job graph that is attached to the
`StreamExecutionEnvironment`. When `env.execute()` is called this graph is packaged up and sent to
the JobManager, which parallelizes the job and distributes slices of it to the Task Managers for
execution. Each parallel slice of your job will be executed in a *task slot*.

Note that if you don't call execute(), your application won't be run.

{{< img src="/fig/distributed-runtime.svg" alt="Flink runtime: client, job manager, task managers" width="80%" >}}

This distributed runtime depends on your application being serializable. It also requires that all
dependencies are available to each node in the cluster.

### Basic stream sources

The example above constructs a `DataStream<Person>` using `env.fromData(...)`. This is a
convenient way to throw together a simple stream for use in a prototype or test.

Another convenient way to get some data into a stream while prototyping is to use a socket

```java
DataStream<String> lines = env.socketTextStream("localhost", 9999);
```

or a file

```java
FileSource<String> fileSource = FileSource.forRecordStreamFormat(
        new TextLineInputFormat(), new Path("file:///path")
    ).build();
DataStream<String> lines = env.fromSource(
    fileSource,
    WatermarkStrategy.noWatermarks(),
    "file-input"
);
```

In real applications the most commonly used data sources are those that support low-latency, high
throughput parallel reads in combination with rewind and replay -- the prerequisites for high
performance and fault tolerance -- such as Apache Kafka, Kinesis, and various filesystems. REST APIs
and databases are also frequently used for stream enrichment.

### Basic stream sinks

The example above uses `adults.print()` to print its results to the task manager logs (which will
appear in your IDE's console, when running in an IDE). This will call `toString()` on each element
of the stream.

The output looks something like this

    1> Fred: age 35
    2> Wilma: age 35

where 1> and 2> indicate which sub-task (i.e., thread) produced the output.

In production, commonly used sinks include the FileSink, various databases,
and several pub-sub systems.

### Debugging

In production, your application will run in a remote cluster or set of containers. And if it fails,
it will fail remotely. The JobManager and TaskManager logs can be very helpful in debugging such
failures, but it is much easier to do local debugging inside an IDE, which is something that Flink
supports. You can set breakpoints, examine local variables, and step through your code. You can also
step into Flink's code, which can be a great way to learn more about its internals if you are
curious to see how Flink works.

{{< top >}}

## Hands-on

At this point you know enough to get started coding and running a simple DataStream application.
Clone the {{< training_repo >}}, and after following the
instructions in the README, do the first exercise: {{< training_link file="/ride-cleansing" name="Filtering a Stream (Ride Cleansing)">}}.

{{< top >}}

## Further Reading

- [Flink Serialization Tuning Vol. 1: Choosing your Serializer — if you can](https://flink.apache.org/news/2020/04/15/flink-serialization-tuning-vol-1.html)
- [Anatomy of a Flink Program]({{< ref "docs/dev/datastream/overview" >}}#anatomy-of-a-flink-program)
- [Data Sources]({{< ref "docs/dev/datastream/overview" >}}#data-sources)
- [Data Sinks]({{< ref "docs/dev/datastream/overview" >}}#data-sinks)
- [DataStream Connectors]({{< ref "docs/connectors/datastream/overview" >}})

{{< top >}}
