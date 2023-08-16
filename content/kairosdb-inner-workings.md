+++
type = "posts"
title = "KairosDB inner workings; how does it store data in Cassandra?"
date = "2017-08-03T16:10:00+03:00"
Categories = ["Database", "Optimization", "Cassandra"]
Tags = ["kairosdb", "cassandra", "schema", "performance", "optimization", "tips" ]
+++

KairosDB is a timeseries database built on Cassandra. Even though it is coded in a way to support different database backends, it is known mostly by its usage of Cassandra as a scaleable and performant time series data storage. With KairosDB, you can store timeseries metric data in different types and then query them with aggregations over time units. In this article, we will look into the details of KairosDB and its two different Cassandra schema to understand how the insert, delete and read operations we do, work under the cover. Cassandra is a database that you need to model your data, regarding to your queries. That is why its important to know how your queries are made to the Cassandra according to your KairosDB data model!

KairosDB has just released a new version that uses CQL and an improved Cassandra Schema this month. This version [reportedly](https://groups.google.com/d/msg/kairosdb-group/V8ViA6RXNZo/7svKXoKcAQAJ) has some performance improvement over the old one. However to be compatible with the old schema, this new version still includes the schema of the old one. Queries are done on the both new and old tables to stay compatible.

The first thing you need to know before inspecting the schemas, is that most of the data in the old schema has a `blob` type. Meaning if you were to query the Cassandra database directly, the returned data would be not human-readable. To get around this when inspecting the database with a GUI or by hand, you can run this Python code in an interpreter to see what the data is: `bytearray.fromhex('7461675f76616c756573').decode('utf8')`. The reason why the data is stored as blobs comes down to custom metric data types and the schema design. We will see this later.

Let's start with dissecting the old Cassandra Schema and see how it works for our read/insert/delete operations.

## Old Cassandra Schema
This Cassandra Schema is used prior to v1.2.0 for the versions of KairosDB that uses Thrift instead of CQL. Check [this gist](https://gist.github.com/Yengas/541808c4ea65d957f052f179f52e6a07) to see the full schema as a gist file.

### Tables
This Schema has a single keyspace called `kairosdb` and 3 tables called `string_index`, `row_key_index` and `data_points`. Each of these tables are explained in detail below.

Metric data for a specific metric and tags are hold in buckets of 3 weeks. This means a new row is created every three weeks to hold metric data for that period of time. The 3 weeks is  [hardcoded](https://github.com/kairosdb/kairosdb/blob/271660f00c009e39e16c1bff536096ec7ed83409/src/main/java/org/kairosdb/datastore/cassandra/CassandraDatastore.java#L92) into KairosDB and is a design choice. Finding out which buckets to query for a period of time, or which bucket a specific data point belongs to is done with the help of [a simple arithmetic calculation](https://github.com/kairosdb/kairosdb/blob/271660f00c009e39e16c1bff536096ec7ed83409/src/main/java/org/kairosdb/datastore/cassandra/CassandraDatastore.java#L684).

#### string_index
This table has [3 fields](https://gist.github.com/Yengas/541808c4ea65d957f052f179f52e6a07#file-cassandra-old-cql-L75) `key = blob`, `column = text` and `value = blob` however only the first two columns are used. KairosDB stores every string it sees on inserts into this table to speed up queries. There are 3 possible values for the `key` field, namely **metric_names**, **tag_names** and **tag_values**(stored as UTF-8 blobs). The other field which is used is the `column` field, which is used to store seen metric names, tag names and tag values. ~~This table is queried to make sure searching the other tables is not unnecessary for a specific metric/tag.~~ This table is queried only when [populating the KairosDB UI](https://groups.google.com/forum/?hl=en#!topic/kairosdb-group/MlwSRmZjhdU)'s metric name dropdown menu. `tag_names` and `tag_values` are not used and they aren't populated in the new version anymore.

#### data_points
This table has [3 fields](https://gist.github.com/Yengas/541808c4ea65d957f052f179f52e6a07#file-cassandra-old-cql-L9) which are used to hold a metric data for a specific metric name and tags. 

The fields for tables are `key = blob`, `column1 = blob` and `value = blob`. The `key` field is the concatenation of metric name, bucket start timestamp, the data type and the tags. `column1` is the offset of this data's time, relative to the bucket start time, and `value` is the serialized binary data for the value(this is how KairosDB can have custom data types) for your metric data. How this columns are utilized are explained below in Workflow section.

The data in this rows are ordered by their offset time in ASC order, so you can have sequential access to your timeseries data when querying. Also another thing to keep in mind is, since the `key` field is used for partioning this data in your Cassandra cluster, each metric data for a specific metric/tags/bucket will be in the same Cassandra node.

#### row_key_index
This table is used to keep track of which buckets are available in the `data_points` table for a specific metric/tags. Lets say you have a time range and you want to do aggregates over a metric with a tag. You need a way of finding matching buckets of data(in `data_points`) for this query first. This tables helps you with this.

It has [3 fields](https://gist.github.com/Yengas/541808c4ea65d957f052f179f52e6a07#file-cassandra-old-cql-L42) `key = blob`, `column1 = blob`, `value = blob`. The `key` field is the metric name in utf-8 bytes. And the `column1` field is the same with the `data_points` table's `key` field. Meaning metric name + bucket start time + data type + tags concatenated together. The `value` field is not used. This table essentially is a way of saying(in plain English); for this metric name, we have these buckets of data with these tags.

You can later use this information to make your queries. This is explained in detail below in Workflow section.

This table's partition key is the `key` field which holds the metric name. Because of this, all of `row_key_index` entries for a single metric get stored in the same cassandra node. This is improved in the new schema.

### Workflow
Below we try to explain how each insert/read and delete operation is done with this schema so you can better understand the inner workings of KairosDB.

As an example in this workflow, we will assume that we're holding temperature data for different cities. Lets model it so the metric name is `Temperature` and the city information is a tag.

#### Insert
Let's say you have a metric data with `Temperature` metric name, and `city=Antalya` tag. For inserting a single metric to this set, we will use `33` for the value and `1501672887988` for the timestamp. 

We first calculate([CassandraDatastore.java Line 287](https://github.com/kairosdb/kairosdb/blob/9469daa937056d84de121ff112a93947e44f960e/src/main/java/org/kairosdb/datastore/cassandra/CassandraDatastore.java#L287)) which bucket this data belongs to. It's done using the formula `timestamp - (timestamp % row_width)` where `row_width` is the hardcoded value([Line 92](https://github.com/kairosdb/kairosdb/blob/develop/src/main/java/org/kairosdb/datastore/cassandra/CassandraDatastore.java#L92)) in KairosDB. This then translates to: `1501672887988 - (1501672887988 % 1814400000) = 1500508800000`. In unix time, for a data dated `08/02/2017 @ 11:21am (UTC)` the bucket start time corresponds to `07/20/2017 @ 12:00am (UTC)`.

We then create([Line 289](https://github.com/kairosdb/kairosdb/blob/9469daa937056d84de121ff112a93947e44f960e/src/main/java/org/kairosdb/datastore/cassandra/CassandraDatastore.java#L289)) a `row_key` with this information. This `row_key` is the concatenation of Metric Name(`Temperature`), the bucket start time we calculated(`1500508800000`), the data type(`kairos_long`) and our tags(`city=Antalya`).

Then we move on to making sure([Line 298](https://github.com/kairosdb/kairosdb/blob/9469daa937056d84de121ff112a93947e44f960e/src/main/java/org/kairosdb/datastore/cassandra/CassandraDatastore.java#L298)) we have a `row_key_index` entry for the Metric Name(`Temperature`) and the `row_key` we calculated. And then we put the data in to the `data_points` table! We do this by calculating the offset relative to the bucket start time `data_time - row_time` in our case: `1501672887988 - 1500508800000 = 1164087988`. This all in all translates to an insert to `data_points` table with `key = row_key`, `column1 = offset we calculated` and the `value = our metric value = 33`.

After the data is written, we write metric names, tag names to the `string_index` table.

#### Read
After getting familiar with insert, we don't need to go into details here. It's apparent that, for a time range and metric/tags query, before querying the `data_points` table, we need to take a look([Line 425](https://github.com/kairosdb/kairosdb/blob/9469daa937056d84de121ff112a93947e44f960e/src/main/java/org/kairosdb/datastore/cassandra/CassandraDatastore.java#L425)) into `row_key_index` table. To do this we create a query into the `row_key_index` table with the start and end time of our query([Line 582](https://github.com/kairosdb/kairosdb/blob/9469daa937056d84de121ff112a93947e44f960e/src/main/java/org/kairosdb/datastore/cassandra/CassandraDatastore.java#L582)). This results in a query with Metric Name(e.g. `Temperature`) and time range to the `row_key_index` table. For each returned result, we match them in-memory with our query([Line 712](https://github.com/kairosdb/kairosdb/blob/9469daa937056d84de121ff112a93947e44f960e/src/main/java/org/kairosdb/datastore/cassandra/CassandraDatastore.java#L712) this requires parsing the binary `row_key` values aswell) and then we make queries([Line 457](https://github.com/kairosdb/kairosdb/blob/9469daa937056d84de121ff112a93947e44f960e/src/main/java/org/kairosdb/datastore/cassandra/CassandraDatastore.java#L457)) to `data_points` table for each matched bucket.

This is why its [not recommended](https://github.com/kairosdb/kairosdb/wiki/Query-Performance) to have a very populated `row_key_index` table with the old schema. Because that may result in too many in-memory comparisons and too many different queries to be made to `data_points` table.
#### Delete
Deletes goes pretty much same with the read. However instead of doing read queries to the `data_points` table, we instead make delete queries. A good part to notice here([Line 522](https://github.com/kairosdb/kairosdb/blob/9469daa937056d84de121ff112a93947e44f960e/src/main/java/org/kairosdb/datastore/cassandra/CassandraDatastore.java#L522)) that, if you happen to delete a whole row of data(meaning the time range you gave includes the start and end time of the bucket), its very performant. However if you do a partial deletes and keep querying that row, you may have performance issues because of [Cassandra tombstones](https://opencredo.com/cassandra-tombstones-common-issues/)(may even result in your read queries to not respond!).

#### Keep in mind!
An important thing to keep in mind when doing reads/deletes is that; Lets say you have 3 metric names and 300.000 cities. To delete 1 week of data for these 3 metrics on Antalya, KairosDB needs to filter all of your `row_key_index` table in-memory and that may translate to millions of comparisons. See calculating `row_key_index` size in Optimization section!

## New Cassandra Schema
The maintainer of the KairosDB database has been doing some work with CQL instead of using THRIFT. The beta version of KairosDB that uses CQL was released in [July 2017](https://github.com/kairosdb/kairosdb/releases/tag/v1.2.0-beta2).

This version has some new tables in the Schema, however its still compatible with the old version of KairosDB, meaning it queries both tables for backwards compatibility. There is a small difference that you need to keep in mind, the new version of the Cassandra schema creates `string_index` table with the `column1` fields as blob. This field was text with the old schema. However these schemas are created only if the table doesn't already exists([Schema.java Line 50](https://github.com/kairosdb/kairosdb/blob/842eb20d258377b3bea156919b6275dfb85f4f04/src/main/java/org/kairosdb/datastore/cassandra/Schema.java#L50)). So it shouldn't cause any problem when upgrading KairosDB to the new version.

The `data_points` and `string_index` are actively used for the same purpose they were used with the old version.

### New Tables
#### row_key_time_index
This table has 3 fields `metric = text`, `row_time = timestamp` and an unused `value = text`. Essentially it's job is to hold what buckets we have for a specific metric. It's queried with the time range and the metric name, and the result is used to query the row_keys_index table.

#### row_keys_index
This table holds the `row_key` we calculated in a CQL friendly way. It has 4 fields namely; `metric = text`, `row_time = timestamp`, `data_type = text` and `tags = frozen<map<text, text>>`. These are used to hold a `row_key` in Cassandra table, so they can be queried with their metric, timestamp and data type to be parsed and compared in memory with the query.

### Workflow
#### Insert
Insert is pretty simple and almost the same as the old schema and addition is that the CQL commands are run batched. Bucket is calculated, then the `row_key` and it's made sure corresponding entries are created in `row_key_time`([Line 54](https://github.com/kairosdb/kairosdb/blob/842eb20d258377b3bea156919b6275dfb85f4f04/src/main/java/org/kairosdb/datastore/cassandra/CQLBatch.java#L54)) and `row_keys`([Line 65](https://github.com/kairosdb/kairosdb/blob/842eb20d258377b3bea156919b6275dfb85f4f04/src/main/java/org/kairosdb/datastore/cassandra/CQLBatch.java#L65)), after this the data is inserted into the `data_points`([Line 116](https://github.com/kairosdb/kairosdb/blob/842eb20d258377b3bea156919b6275dfb85f4f04/src/main/java/org/kairosdb/datastore/cassandra/CQLBatch.java#L116)) and only the metric names are inserted into the `string_index`.
#### Read
We still need a way to find which `data_points` buckets we need to query. To do this, KairosDB needs to query both old schema's `row_key_index` table and the new `row_key_time_index` and `row_keys` table.

The old schema is queried like the old version of KairosDB but instead of using Thrift, CQL is used.

For the new schema; `row_key_time_index` is queried with the metric name and the time range([Line 780](https://github.com/kairosdb/kairosdb/blob/842eb20d258377b3bea156919b6275dfb85f4f04/src/main/java/org/kairosdb/datastore/cassandra/CassandraDatastore.java#L780)) to get the buckets available, then multiple queries are made to the `row_keys` table([Line 788](https://github.com/kairosdb/kairosdb/blob/842eb20d258377b3bea156919b6275dfb85f4f04/src/main/java/org/kairosdb/datastore/cassandra/CassandraDatastore.java#L788)) and the returning entries are filtered in memory([Line 823](https://github.com/kairosdb/kairosdb/blob/842eb20d258377b3bea156919b6275dfb85f4f04/src/main/java/org/kairosdb/datastore/cassandra/CassandraDatastore.java#L823)) according to the query made.
#### Delete
New version of cassandra [doesn't seem to delete](https://github.com/kairosdb/kairosdb/blob/842eb20d258377b3bea156919b6275dfb85f4f04/src/main/java/org/kairosdb/datastore/cassandra/CassandraDatastore.java#L595) `row_key_time_index` and `row_keys` entries for the metric data. It just deletes the old schema's `row_key_index` entry, and the `data_points` entries([Line 595](https://github.com/kairosdb/kairosdb/blob/842eb20d258377b3bea156919b6275dfb85f4f04/src/main/java/org/kairosdb/datastore/cassandra/CassandraDatastore.java#L595) and [Line 614](https://github.com/kairosdb/kairosdb/blob/842eb20d258377b3bea156919b6275dfb85f4f04/src/main/java/org/kairosdb/datastore/cassandra/CassandraDatastore.java#L614)). There are no delete queries for the new tables in [Schema.java](https://github.com/kairosdb/kairosdb/blob/842eb20d258377b3bea156919b6275dfb85f4f04/src/main/java/org/kairosdb/datastore/cassandra/Schema.java).

## Optimization, what to keep in mind
### Calculating your row_key_index size
To do this calculation. Lets say you have 300.000 different cities, and you hold `Temperature`, `Humidity` and `Wind` metrics for these cities. And you decided to model your data in KairosDB so the metric names correspond to the `Temperature`, `Humidity` and `Wind`. And the cities are stored with the `city=XXX` tag. You will have `N = 300.000 * 3 = 900.000` entries in your `row_key_index`. And you get `N` more `row_key_index` entries each 3 week because KairosDB divides your data into buckets! You can see how this could slow your queries with the Read and Delete, because of in-memory comparisons. The situation is different with the new schema. 

### Gotchas, not many performance improvements... Slow deletes, reads.
KairosDB isn't the most optimized database when it comes to executing your queries on Cassandra. One of the most apparent feature it lacks, is that when you make a query with same metric name, but different tags, it queries the `row_key_index` table multiple times. And considering this metric name is populated, this may cause a lot of slow downs on your system. This can happen both with reads and deletes.

Lets keep going with our example where we had 3 different metrics `Temperature`, `Humidity` and `Wind` and we had 1 tag named city, which holds the city name. If we were to run a delete query which deletes all of the metrics for 2 different cities `Antalya` and `Istanbul`, this will cause 2 scans on all of our `row_key_index`. Lets think a scenario where we run this kind of query for deleting a big number of cities... This could cause a serious slowdown with the current schema(s) if you were to make a naive query with multiple metrics. To make this query more performant and make it read the `row_key_index` table once, try using group by as mentioned by Brian Hawkins on [this kairosdb group post](https://groups.google.com/d/msg/kairosdb-group/MlwSRmZjhdU/WndUNMi1AwAJ). Example query:

```
{
    "start_absolute": ...,
    "end_absolute": ...,
    "metrics": [{ 
        "name": "Temperature",
        "tags": { "city": ["Antalya", "Istanbul"] }, 
        "group_by": [
            {
              "name": "tag",
              "tags": [ "city" ]
            }
        ]
    }]
}
```

### Don't have too many tombstones
Cassandra doesn't delete records when you execute a delete query. It just marks them as tombstones and deletes them in the next compaction. Until this compaction happens, every query you make to these sstables, will need to filter out tombstones records and this may cause problem such as your read queries not responding or getting slower. You should keep in mind KairosDB stores 3 weeks of metric data in a row for each specific metric/tag combinations. If you delete some data in the row you are actively using, you may get slow downs/halts/aborts. Please see this [stackoverflow question](https://stackoverflow.com/questions/21755286/what-exactly-happens-when-tombstone-limit-is-reached), regarding this problem.
