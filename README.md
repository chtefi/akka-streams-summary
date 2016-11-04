# Akka Streams: A stream processing library

As any streaming library, we have the concept of
- Source (input): create messages
- Channel (flow/stage) (more globally, Shape here): messages pass through
- Sink (output): output messages somewhere else (Note: it exists a Sink.ignore to output to a black-hole)

In akka-streams, we have also these abstractions:

- Outlet[T] related to a Source[T]: .out on the Source
- Inlet[T] related to a Sink[T]: .in on the Sink

## Shapes

A Shape is a "box" with inputs and outputs, something that "processes" messages. There are some specific kind of shapes:

- Shape: blackbox without inputs (inlets), and outputs (outlets)
- CloseShape: Shape with closed inputs and closed outputs (can be materialized)
- SourceShape: Shape with 1 input only
- FlowShape: Shape with 1 input, 1 output
- SinkShape: Shape with 1 output only
- BibiShape: Shape with 2 inputs, 2 outputs

## Graph

A Graph is a pipeline. It combines inputs, flows, and outputs.

- ActorMaterializer: AkkaStreams specific. Provisions actors to execute a pipeline (a Graph)
- ActorMaterializerSettings: ActorMaterializer ... settings. Can have a custom supervision strategy (if exception, Resume, Restart, or Stop), can add logging, can configure thresholds..
- RunnableGraph: A graph is a whole set of Outlet/FlowShape/Inlet linked together.
 - it starts with a Source (Outlet), ends with a Sink (Inlet)
- It can be stopped anytime using a KillSwitch.

```scala
// this is a Graph that can be run
Source.fromFuture(Future { 2 })
    .via(Flow[Int].map(_ * 2))
    .to(Sink.foreach(println))
    
// this is the same with the full DSL
val g: RunnableGraph[_] = RunnableGraph.fromGraph(GraphDSL.create() {
    implicit builder =>
      val source = builder.add(Source.fromFuture(Future { 2 }))
      val multiply = builder.add(Flow[Int].map(_ * 2))
      val sink = builder.add(Sink.foreach(println))

      import GraphDSL.Implicits._
      source ~> multiply ~> sink
      ClosedShape
  })
```

## Flow 

A flow can be 
- 1 -> 1
- 1 -> N: faning out events (Broadcast) or acts as a load balancer (Balance).
- N -> 1: merge 1 event of several inputs into 1 event in output; or simply Concat (3 inputs = 3 outputs).

A Flow is a Graph.

```scala
// this is a Flow: Int -> Int
Flow[Int].map(_ * 2)
```

## Source

There are a lot of way (syntaxes) to execute a pipeline in Akka Streams.
For instance, a Source can be ran without Graph, without Flow (it's created underneath):
```scala
Source(0 to 5).map(100 * _).runWith(Sink.fold(0)(_ + _)) // returns a Future[Int]
FileIO.fromPath(Paths.get("log.txt")).runFold(0)(_ + _.length).foreach(println)
```

## Composition

We can compose anything using `.via`, like a (source+flow) => 1x source, 2x flows => 1x flow

## Using Actors

It's possible to bind the Source to a custom Actor.
akka-streams will send requests asking for `n` elements. The actor can then call `n` times `onNext` (not more or an exception will be thrown):
```scala
class Toto extends Actor with ActorPublisher[Char] {
  override def receive = {
    case x @ Request(n) => println(x); (0 until n.toInt).foreach(_ => onNext(Random.nextPrintableChar()))
    case Cancel => context.stop(self)
  }
}

Source.actorPublisher(Props(classOf[Toto])).runForeach(println)
```

## Kafka

[reactive-kafka](https://github.com/akka/reactive-kafka) === akka-stream-kafka

## Alpakka

Set of connectors useable with Akka Streams:

- HTTP
- TCP
- File IO

# Reactive streams: A specification

- Subscriber[A] (Consumer), Publisher[B] (Producer), Subscription, Processor[A, B]
- asynchronous boundary: decouple components
- define backpressure model: signal to the source to handle
- push AND pull: fast consumer AND slow consumer (dynamic)
- ability to batch process

Some implementations are:
- RxJava
- Akka-streams
- Reactor
- Vert.x
- Slick



# Reactive Programming

- Responsive -> Resilient + Scalable -> Message Driven

# References

- https://medium.com/@kvnwbbr/diving-into-akka-streams-2770b3aeabb0 
- https://medium.com/reactive-programming/what-is-reactive-programming-bc9fa7f4a7fc
- https://medium.com/@kvnwbbr/a-journey-into-reactive-streams-5ee2a9cd7e29

