---
description: "Akka.Streams for data pipelines — Source/Flow/Sink composition, MergeHub/BroadcastHub, StreamRefs, custom stages, backpressure, and stream supervision"
---

# Akka.Streams

Reactive stream pipelines for Akka.NET: backpressured, composable, and integrated
with actors. Use streams for data flow; use actors for lifecycle and state.

## When to use

- Data pipelines: poll → transform → enrich → publish
- Fan-out/fan-in patterns (one source, many consumers or vice versa)
- Rate limiting / throttling external API calls
- Bridging async producers/consumers with backpressure
- Any pipeline where you'd otherwise chain actors just to pass data through

## Core concepts

```
Source<T, TMat>  →  Flow<TIn, TOut, TMat>  →  Sink<T, TMat>
   (produces)          (transforms)              (consumes)
```

- **Source** — produces elements (timer tick, database query, actor messages)
- **Flow** — transforms elements (map, filter, buffer, throttle, group)
- **Sink** — consumes elements (actor Tell, HTTP publish, file write, aggregation)
- **Materialized value (TMat)** — runtime handle produced when the stream starts (e.g. a cancellation token, a completion Task)

## Basic pipeline

```csharp
using Akka.Streams;
using Akka.Streams.Dsl;

var materializer = Context.Materializer();

Source.Tick(TimeSpan.Zero, TimeSpan.FromMinutes(60), "tick")
    .Select(_ => new PollRequest(locations))
    .SelectAsync(1, req => FetchForecast(req))
    .Where(result => result.IsSuccess)
    .Select(result => Enrich(result.Data))
    .To(Sink.ForEach<EnrichedForecast>(data => PublishToMqtt(data)))
    .Run(materializer);
```

### Materializer

Every stream needs a materializer to run. In actors, use `Context.Materializer()`.
Reuse a single materializer per actor — creating one is lightweight but unnecessary
to repeat.

```csharp
public sealed class PipelineActor : ReceiveActor
{
    private readonly IMaterializer _materializer;

    public PipelineActor()
    {
        _materializer = Context.Materializer();
        // ... build and run streams
    }
}
```

## Throttling external APIs

Rate-limit requests to respect API budgets:

```csharp
Source.From(modelIds)
    .Throttle(10, TimeSpan.FromSeconds(1), 1, ThrottleMode.Shaping)
    .SelectAsync(4, async modelId =>
    {
        var response = await httpClient.GetAsync($"/forecast?model={modelId}");
        return await response.Content.ReadFromJsonAsync<Forecast>();
    })
    .To(Sink.ActorRef<Forecast>(processorActor, new StreamCompleted()))
    .Run(_materializer);
```

`Throttle` parameters:
- `elements` — max elements per period
- `per` — time period
- `maximumBurst` — burst above steady rate
- `ThrottleMode.Shaping` — delays elements; `Enforcing` — drops on overflow

## Fan-out: one source, many consumers

### BroadcastHub — dynamic fan-out

```csharp
// Producer side: create a hub that broadcasts to all connected consumers
var (sink, source) = MergeHub.Source<ForecastUpdate>()
    .Via(enrichmentFlow)
    .ToMaterialized(BroadcastHub.Sink<EnrichedForecast>(), Keep.Both)
    .Run(_materializer);

// Any consumer can tap in:
source.RunWith(Sink.ForEach<EnrichedForecast>(
    update => mqttActor.Tell(update)), _materializer);

source.RunWith(Sink.ForEach<EnrichedForecast>(
    update => metricsActor.Tell(update)), _materializer);
```

### Static Broadcast (fixed fan-out)

```csharp
var graph = GraphDsl.Create(builder =>
{
    var broadcast = builder.Add(new Broadcast<ForecastData>(2));
    var source = builder.Add(Source.From(forecasts));
    var mqttSink = builder.Add(Sink.ForEach<ForecastData>(PublishMqtt));
    var metricsSink = builder.Add(Sink.ForEach<ForecastData>(RecordMetrics));

    builder.From(source).To(broadcast);
    builder.From(broadcast.Out(0)).To(mqttSink);
    builder.From(broadcast.Out(1)).To(metricsSink);

    return ClosedShape.Instance;
});

RunnableGraph.FromGraph(graph).Run(_materializer);
```

## Fan-in: many sources, one consumer

### MergeHub — dynamic fan-in

```csharp
// Consumer side: create a hub that accepts elements from any producer
var sink = MergeHub.Sink<ModelForecast>()
    .GroupBy(20, f => f.CycleId)
    .Fold(AggregatedForecast.Empty, (agg, f) => agg.Add(f))
    .MergeSubstreams()
    .To(Sink.ActorRef<AggregatedForecast>(consensusActor, StreamCompleted.Instance))
    .Run(_materializer);

// Any producer can push:
foreach (var modelId in modelIds)
{
    Source.Single(new ModelForecast(modelId, data))
        .RunWith(sink, _materializer);
}
```

### Static Merge

```csharp
var merged = Source.From(sourceA)
    .Merge(Source.From(sourceB))
    .Merge(Source.From(sourceC));
```

## StreamRefs — streams across actors

Pass a stream endpoint to another actor as a message. The stream runs across
actor boundaries with full backpressure.

### SinkRef — send a "write endpoint" to a producer

```csharp
// Consumer actor: create a SinkRef and send it to the producer
var sinkRef = StreamRefs.SinkRef<DataChunk>()
    .To(Sink.ForEach<DataChunk>(chunk => Process(chunk)))
    .Run(_materializer);

producerActor.Tell(new StartStreaming(sinkRef));

// Producer actor: connect a source to the received SinkRef
Receive<StartStreaming>(msg =>
{
    Source.From(dataChunks)
        .RunWith(msg.SinkRef.Sink, _materializer);
});
```

### SourceRef — send a "read endpoint" to a consumer

```csharp
// Producer: create a SourceRef and send it
var sourceRef = Source.From(dataChunks)
    .RunWith(StreamRefs.SourceRef<DataChunk>(), _materializer);

consumerActor.Tell(new DataReady(sourceRef));

// Consumer: connect the received SourceRef to a sink
Receive<DataReady>(msg =>
{
    msg.SourceRef.Source
        .RunWith(Sink.ForEach<DataChunk>(Process), _materializer);
});
```

## Custom GraphStage

For stream logic that can't be expressed with built-in operators.

```csharp
public sealed class BudgetThrottleStage : GraphStage<FlowShape<ApiRequest, ApiRequest>>
{
    private readonly int _maxPerMinute;

    public BudgetThrottleStage(int maxPerMinute)
    {
        _maxPerMinute = maxPerMinute;
        Shape = new FlowShape<ApiRequest, ApiRequest>(In, Out);
    }

    public Inlet<ApiRequest> In { get; } = new("BudgetThrottle.in");
    public Outlet<ApiRequest> Out { get; } = new("BudgetThrottle.out");

    public override FlowShape<ApiRequest, ApiRequest> Shape { get; }

    protected override GraphStageLogic CreateLogic(Attributes inheritedAttributes) =>
        new Logic(this);

    private sealed class Logic : GraphStageLogic
    {
        private int _count;
        private readonly BudgetThrottleStage _stage;

        public Logic(BudgetThrottleStage stage) : base(stage.Shape)
        {
            _stage = stage;

            SetHandler(stage.In, onPush: () =>
            {
                var element = Grab(stage.In);
                if (_count < _stage._maxPerMinute)
                {
                    _count++;
                    Push(stage.Out, element);
                }
                else
                {
                    // budget exceeded — drop or backpressure
                    Pull(stage.In);
                }
            });

            SetHandler(stage.Out, onPull: () => Pull(stage.In));
        }
    }
}

// Usage in pipeline:
source.Via(new BudgetThrottleStage(600)).To(sink);
```

## Stream supervision

Handle element-level failures without stopping the entire stream:

```csharp
var decider = new DeciderBuilder()
    .Match<HttpRequestException>(Directive.Resume)   // skip failed HTTP calls
    .Match<JsonException>(Directive.Resume)           // skip parse errors
    .MatchAny(Directive.Stop)                         // stop on unknown errors
    .Build();

Source.From(requests)
    .SelectAsync(4, FetchAsync)
    .WithAttributes(ActorAttributes.CreateSupervisionStrategy(decider))
    .To(Sink.ForEach<Result>(Process))
    .Run(_materializer);
```

Directives:
- `Resume` — drop the failing element, continue with the next
- `Restart` — drop the element AND reset stage state, continue
- `Stop` — stop the entire stream (default)

## Actor ↔ Stream integration

### Sink.ActorRef — stream → actor

```csharp
source.RunWith(
    Sink.ActorRef<ProcessedData>(targetActor, new StreamCompleted()),
    _materializer);
```

The actor receives each stream element as a message. `StreamCompleted` is sent
when the stream finishes normally.

### Source.ActorRef — actor → stream

```csharp
var actorRef = Source.ActorRef<DataItem>(
        bufferSize: 100,
        overflowStrategy: OverflowStrategy.DropHead)
    .To(Sink.ForEach<DataItem>(Process))
    .Run(_materializer);

// Now any actor can push elements into the stream:
actorRef.Tell(new DataItem("value"));
actorRef.Tell(new Status.Success("complete")); // completes the stream
```

### Source.Queue — programmatic push with backpressure

```csharp
var (queue, done) = Source.Queue<WorkItem>(100, OverflowStrategy.Backpressure)
    .SelectAsync(4, ProcessAsync)
    .ToMaterialized(Sink.Ignore<Result>(), Keep.Both)
    .Run(_materializer);

// Offer elements with backpressure feedback:
var result = await queue.OfferAsync(new WorkItem(data));
// result: Enqueued, Dropped, QueueClosed, or Failure
```

## Reusable Flows

Extract reusable transformations as `Flow<TIn, TOut, NotUsed>`:

```csharp
public static class EnrichmentFlows
{
    public static Flow<RawForecast, EnrichedForecast, NotUsed> Enrichment(
        IConsensusCalculator consensus, IAlertEvaluator alerts) =>
        Flow.Create<RawForecast>()
            .Select(f => consensus.Calculate(f))
            .Select(f => alerts.Evaluate(f))
            .Where(f => f.IsValid);
}

// Compose:
source.Via(EnrichmentFlows.Enrichment(consensus, alerts)).To(sink);
```

## Checklist

- [ ] One `IMaterializer` per actor via `Context.Materializer()`
- [ ] `Throttle` for external API rate limiting
- [ ] `SelectAsync(parallelism, ...)` for concurrent async operations within a stage
- [ ] Stream supervision strategy for element-level error handling
- [ ] `Sink.ActorRef` to bridge stream output into actor messages
- [ ] `MergeHub` / `BroadcastHub` for dynamic fan-in/fan-out
- [ ] `StreamRefs` for passing stream endpoints across actors
- [ ] Custom `GraphStage` only when built-in operators aren't sufficient
- [ ] Reusable `Flow<TIn, TOut, NotUsed>` for shared transformations
