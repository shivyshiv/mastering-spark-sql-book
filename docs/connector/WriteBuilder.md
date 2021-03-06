# WriteBuilder

`WriteBuilder` is an [abstraction](#contract) of [write builders](#implementations) for [batch](#buildForBatch) and [streaming](#buildForStreaming).

## Contract

### <span id="buildForBatch"> Creating BatchWrite

```java
BatchWrite buildForBatch()
```

[BatchWrite](BatchWrite.md) for writing data to a batch source

Used when:

* [CreateTableAsSelectExec](../physical-operators/CreateTableAsSelectExec.md), [ReplaceTableAsSelectExec](../physical-operators/ReplaceTableAsSelectExec.md), [AppendDataExec](../physical-operators/AppendDataExec.md), [OverwriteByExpressionExec](../physical-operators/OverwriteByExpressionExec.md), [OverwritePartitionsDynamicExec](../physical-operators/OverwritePartitionsDynamicExec.md), [AtomicTableWriteExec](../physical-operators/AtomicTableWriteExec.md) physical commands are executed

### <span id="buildForStreaming"> Creating StreamingWrite

```java
StreamingWrite buildForStreaming()
```

`StreamingWrite` for writing data to a streaming source

Used when:

* `StreamExecution` stream execution engine (Spark Structured Streaming) is requested to `createStreamingWrite`

## Implementations

* ConsoleTable (Spark Structured Streaming)
* ForeachWriterTable (Spark Structured Streaming)
* [KafkaTable](../datasources/kafka/KafkaTable.md)
* MemorySink (Spark Structured Streaming)
* [FileWriteBuilder](../FileWriteBuilder.md)
* [NoopWriteBuilder](../datasources/noop/NoopWriteBuilder.md)
* [SupportsDynamicOverwrite](SupportsDynamicOverwrite.md)
* [SupportsOverwrite](SupportsOverwrite.md)
* [SupportsStreamingUpdate](SupportsStreamingUpdate.md)
* [SupportsTruncate](SupportsTruncate.md)
* [V1WriteBuilder](V1WriteBuilder.md)
