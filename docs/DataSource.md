# DataSource &mdash; Pluggable Data Provider Framework

`DataSource` paves the way for **Pluggable Data Provider Framework** in Spark SQL.

Together with the [provider interfaces](#provider-interfaces), `DataSource` allows Spark SQL integrators to use external data systems as data sources and sinks in structured queries in Spark SQL (incl. Spark Structured Streaming).

## Provider Interfaces

* [CreatableRelationProvider](CreatableRelationProvider.md)
* [FileFormat](datasources/FileFormat.md)
* [RelationProvider](RelationProvider.md)
* [SchemaRelationProvider](SchemaRelationProvider.md)
* StreamSinkProvider
* StreamSourceProvider

## Accessing DataSource

`DataSource` is available using [DataFrameReader](DataFrameReader.md).

```text
val people = spark
  .read // Batch reading
  .format("csv")
  .load("people.csv")
```

```text
val messages = spark
  .readStream // Streamed reading
  .format("kafka")
  .option("subscribe", "topic")
  .option("kafka.bootstrap.servers", "localhost:9092")
  .load
```

## Creating Instance

`DataSource` takes the following to be created:

* <span id="sparkSession"> [SparkSession](SparkSession.md)
* <span id="className"> Fully-qualified class name or an alias of the data source provider (aka _data source format_)
* <span id="paths"> Data Paths (default: empty)
* <span id="userSpecifiedSchema"> User-specified [schema](StructType.md) (default: undefined)
* <span id="partitionColumns"> Names of the partition columns (default: empty)
* <span id="bucketSpec"> [Bucketing specification](spark-sql-BucketSpec.md) (default: undefined)
* <span id="options"> Options (default: empty)
* <span id="catalogTable"> [CatalogTable](CatalogTable.md) (default: undefined)

!!! note
    Only a [SparkSession](#sparkSession) and a [fully-qualified class name of the data source provider](#className) are required to create an instance of `DataSource`.

`DataSource` is created when:

* `HiveMetastoreCatalog` is requested to [convert a HiveTableRelation to a LogicalRelation over a HadoopFsRelation](hive/HiveMetastoreCatalog.md#convertToLogicalRelation)

* `DataFrameReader` is requested to [load data from a data source (Data Source V1)](DataFrameReader.md#loadV1Source)

* `DataFrameWriter` is requested to [save to a data source (Data Source V1)](DataFrameWriter.md#saveToV1Source)

* [CreateDataSourceTableCommand](logical-operators/CreateDataSourceTableCommand.md), [CreateDataSourceTableAsSelectCommand](logical-operators/CreateDataSourceTableAsSelectCommand.md), [InsertIntoDataSourceDirCommand](logical-operators/InsertIntoDataSourceDirCommand.md), [CreateTempViewUsing](logical-operators/CreateTempViewUsing.md) commands are executed

* [FindDataSourceTable](logical-analysis-rules/FindDataSourceTable.md) and [ResolveSQLOnFile](logical-analysis-rules/ResolveSQLOnFile.md) logical evaluation rules are executed

* For Spark Structured Streaming's `FileStreamSource`, `DataStreamReader` and `DataStreamWriter`

## Data Source Resolution

`DataSource` is given an [alias or a fully-qualified class name](#className) of the data source provider. `DataSource` uses the name to [load the Java class](#lookupDataSource). In the end, `DataSource` uses the Java class to [resolve a relation](#resolveRelation) to represent the data source in logical plans.

## <span id="lookupDataSource"> Loading Java Class Of Data Source Provider

```scala
lookupDataSource(
  provider: String,
  conf: SQLConf): Class[_]
```

`lookupDataSource` first finds the given `provider` in the [backwardCompatibilityMap](#backwardCompatibilityMap) internal registry, and falls back to the `provider` name itself when not found.

!!! note
    The `provider` argument can be either an alias (a simple name, e.g. `parquet`) or a fully-qualified class name (e.g. `org.apache.spark.sql.execution.datasources.parquet.ParquetFileFormat`).

`lookupDataSource` then uses the given [SQLConf](SQLConf.md) to decide on the class name of the provider for ORC and Avro data sources as follows:

* For `orc` provider and [native](SQLConf.md#ORC_IMPLEMENTATION), `lookupDataSource` uses the new ORC file format [OrcFileFormat](datasources/orc/OrcFileFormat.md) (based on Apache ORC)

* For `orc` provider and [hive](SQLConf.md#ORC_IMPLEMENTATION), `lookupDataSource` uses `org.apache.spark.sql.hive.orc.OrcFileFormat`

* For `com.databricks.spark.avro` and [spark.sql.legacy.replaceDatabricksSparkAvro.enabled](SQLConf.md#replaceDatabricksSparkAvroEnabled) configuration enabled (default), `lookupDataSource` uses the built-in (but external) [Avro data source](datasources/avro/AvroFileFormat.md) module

`lookupDataSource` uses `DefaultSource` as the class name as another provider name variant (i.e. `[provider1].DefaultSource`).

`lookupDataSource` uses Java's [ServiceLoader]({{ java.api }}/java/util/ServiceLoader.html) service-provider loading facility to find all data source providers of type [DataSourceRegister](DataSourceRegister.md) on the Spark CLASSPATH.

`lookupDataSource` tries to find the `DataSourceRegister` provider classes (by their [alias](DataSourceRegister.md#shortName)) that match the provider name (case-insensitive, e.g. `parquet` or `kafka`).

If a single `DataSourceRegister` provider class is found, `lookupDataSource` simply returns the instance of the data source provider.

If no `DataSourceRegister` provider class could be found by the short name (alias), `lookupDataSource` tries to load the provider name to be a fully-qualified class name. If not successful, `lookupDataSource` tries to load the other provider name (aka _DefaultSource_) instead.

!!! note
    [DataFrameWriter.format](DataFrameWriter.md#format) and [DataFrameReader.format](DataFrameReader.md#format) methods accept the name of the data source provider to use as an alias or a fully-qualified class name.

```text
import org.apache.spark.sql.execution.datasources.DataSource
val source = "parquet"
val cls = DataSource.lookupDataSource(source, spark.sessionState.conf)
```

`lookupDataSource` is used when:

* `SparkSession` is requested to [executeCommand](SparkSession.md#executeCommand)
* `CreateTableLikeCommand` and `AlterTableAddColumnsCommand` runnable commands are executed
* `DataSource` is requested for [providingClass](#providingClass) and to [lookupDataSourceV2](#lookupDataSourceV2)
* [PreprocessTableCreation](logical-analysis-rules/PreprocessTableCreation.md) posthoc logical resolution rule is executed
* `DataStreamReader` ([Spark Structured Streaming]({{ book.structured_streaming }}/DataStreamReader)) is requested to `load`
* `DataStreamWriter` ([Spark Structured Streaming]({{ book.structured_streaming }}/DataStreamWriter)) is requested to `start` a streaming query

## <span id="resolveRelation"> Resolving Relation

```scala
resolveRelation(
  checkFilesExist: Boolean = true): BaseRelation
```

`resolveRelation` resolves (creates) a [BaseRelation](BaseRelation.md).

Internally, `resolveRelation` creates an instance of the [providingClass](#providingClass) and branches based on type and whether the [user-defined schema](#userSpecifiedSchema) was specified or not.

.Resolving BaseRelation per Provider and User-Specified Schema
[cols="1,3",options="header",width="100%"]
|===
| Provider
| Behaviour

| [SchemaRelationProvider](SchemaRelationProvider.md)
| Executes [SchemaRelationProvider.createRelation](SchemaRelationProvider.md#createRelation) with the provided schema

| RelationProvider.md[RelationProvider]
| Executes RelationProvider.md#createRelation[RelationProvider.createRelation]

| [FileFormat](datasources/FileFormat.md)
| Creates a [HadoopFsRelation](BaseRelation.md#HadoopFsRelation)
|===

`resolveRelation` is used when:

* `DataSource` is requested to <<writeAndRead, write and read>> the result of a structured query (only when <<providingClass, providingClass>> is a [FileFormat](datasources/FileFormat.md))

* `DataFrameReader` is requested to [load data from a data source that supports multiple paths](DataFrameReader.md#load)

* `TextInputCSVDataSource` and `TextInputJsonDataSource` are requested to infer schema

* `CreateDataSourceTableCommand` runnable command is CreateDataSourceTableCommand.md#run[executed]

* `CreateTempViewUsing` logical command is requested to <<CreateTempViewUsing.md#run, run>>

* `FindDataSourceTable` is requested to [readDataSourceTable](logical-analysis-rules/FindDataSourceTable.md#readDataSourceTable)

* `ResolveSQLOnFile` is requested to convert a logical plan (when <<providingClass, providingClass>> is a [FileFormat](datasources/FileFormat.md))

* `HiveMetastoreCatalog` is requested to hive/HiveMetastoreCatalog.md#convertToLogicalRelation[convert a HiveTableRelation to a LogicalRelation over a HadoopFsRelation]

* Structured Streaming's `FileStreamSource` creates batches of records

## <span id="planForWriting"> Creating Logical Command for Writing (for CreatableRelationProvider and FileFormat Data Sources)

```scala
planForWriting(
  mode: SaveMode,
  data: LogicalPlan): LogicalPlan
```

`planForWriting` creates an instance of the [providingClass](#providingClass) and branches off per type as follows:

* For a [CreatableRelationProvider](CreatableRelationProvider.md), `planForWriting` creates a <<SaveIntoDataSourceCommand.md#creating-instance, SaveIntoDataSourceCommand>> (with the input `data` and `mode`, the `CreatableRelationProvider` data source and the <<caseInsensitiveOptions, caseInsensitiveOptions>>)

* For a [FileFormat](datasources/FileFormat.md), `planForWriting` [planForWritingFileFormat](#planForWritingFileFormat) (with the `FileFormat` format and the input `mode` and `data`)

* For other types, `planForWriting` simply throws a `RuntimeException`:

    ```text
    [providingClass] does not allow create table as select.
    ```

`planForWriting` is used when:

* `DataFrameWriter` is requested to [save (to a data source V1](DataFrameWriter.md#saveToV1Source)
* [InsertIntoDataSourceDirCommand](logical-operators/InsertIntoDataSourceDirCommand.md) logical command is executed

## <span id="writeAndRead"> Writing Data to Data Source (per Save Mode) Followed by Reading Rows Back (as BaseRelation)

```scala
writeAndRead(
  mode: SaveMode,
  data: LogicalPlan,
  outputColumnNames: Seq[String],
  physicalPlan: SparkPlan): BaseRelation
```

`writeAndRead`...FIXME

!!! note
    `writeAndRead` is also known as **Create Table As Select** (CTAS) query.

`writeAndRead` is used when [CreateDataSourceTableAsSelectCommand](logical-operators/CreateDataSourceTableAsSelectCommand.md) logical command is executed.

## <span id="planForWritingFileFormat"> Planning for Writing (to FileFormat-Based Data Source)

```scala
planForWritingFileFormat(
  format: FileFormat,
  mode: SaveMode,
  data: LogicalPlan): InsertIntoHadoopFsRelationCommand
```

`planForWritingFileFormat` takes the [paths](#paths) and the `path` option (from the [caseInsensitiveOptions](#caseInsensitiveOptions)) together and (assuming that there is only one path available among the paths combined) creates a fully-qualified HDFS-compatible output path for writing.

!!! note
    `planForWritingFileFormat` uses Hadoop HDFS's Hadoop [Path]({{ hadoop.api }}/org/apache/hadoop/fs/Path.html) to requests for the [FileSystem]({{ hadoop.api }}/org/apache/hadoop/fs/FileSystem.html) that owns it (using a [Hadoop Configuration](SessionState.md#newHadoopConf)).

`planForWritingFileFormat` [validates partition columns](spark-sql-PartitioningUtils.md#validatePartitionColumn) in the given [partitionColumns](#partitionColumns).

In the end, `planForWritingFileFormat` returns a new [InsertIntoHadoopFsRelationCommand](logical-operators/InsertIntoHadoopFsRelationCommand.md).

`planForWritingFileFormat` throws an `IllegalArgumentException` when there are more than one [path](#paths) specified:

```text
Expected exactly one path to be specified, but got: [allPaths]
```

`planForWritingFileFormat` is used when `DataSource` is requested for the following:

* [Writing data to a data source followed by "reading" rows back](#writeAndRead) (for [CreateDataSourceTableAsSelectCommand](logical-operators/CreateDataSourceTableAsSelectCommand.md) logical command)

* [Creating a logical command for writing](#planForWriting) (for [InsertIntoDataSourceDirCommand](logical-operators/InsertIntoDataSourceDirCommand.md) logical command and [DataFrameWriter.save](DataFrameWriter.md#save) operator with DataSource V1 data sources)

## <span id="providingClass"> Data Source Class

```scala
providingClass: Class[_]
```

[java.lang.Class]({{ java.api }}/java/lang/Class.html) that was [loaded](#lookupDataSource) for the given [data source provider](#className)

`providingClass` is used when:

* [InsertIntoDataSourceDirCommand](logical-operators/InsertIntoDataSourceDirCommand.md) logical command is executed (to ensure working with a [FileFormat](datasources/FileFormat.md)-based data source)
* [ResolveSQLOnFile](logical-analysis-rules/ResolveSQLOnFile.md) logical evaluation rule is executed (to ensure working with a [FileFormat](datasources/FileFormat.md)-based data source)
* `DataSource` is requested for [providingInstance](#providingInstance)

## <span id="providingInstance"> Data Source Instance

```scala
providingInstance(): Any
```

`providingInstance` simply creates an instance of the [Java class](#providingClass) of the data source.

`providingInstance` is used when:

* `DataSource` is requested to `sourceSchema`, `createSource`, `createSink`, [resolve a relation](#resolveRelation), [write and read](#writeAndRead) and [plan for writing](#planForWriting)
