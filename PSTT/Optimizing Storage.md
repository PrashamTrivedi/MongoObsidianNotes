---
alias: DF200
---
# Explaining Queries.

- Can explain the query performance by using `.explain()` method on query or cursor. 
	- In this way default mode is `queryPlanner` mode.
- The other way is use `db.runCommand({explain:{query}})` 
	- In this way default mode is `allPlansExecution` 

## Explain modes

In all modes MongoDB runs [Query Optimizer](https://www.mongodb.com/docs/v6.0/core/query-plans/) to chose winning plan. What happens after that is where modes differ.
### Query Planner mode
- Value `queryPlanner`
- Shows winning plan but does not execute actual query.
- Default for `db.collection.find().execute()` modes.
- Returns only [QueryPlanner](https://www.mongodb.com/docs/v6.0/reference/explain-results/#queryplanner) data
### Execution Stats mode
- Value `executionStats`
- Executes actual query and returns statistics regarding to the execution.
- For write operations this mode does not actually execute any writes.
- Returns [QueryPlanner](https://www.mongodb.com/docs/v6.0/reference/explain-results/#queryplanner) and [ExecutionStats](https://www.mongodb.com/docs/v6.0/reference/explain-results/#executionstats) objects
### All Plans Execution mode
- Value `allPlansExecution`
- Executes actual query and winning plan.
- For write operations this mode does not actually execute any writes
- This mode also mentions all stats about other plans.
- Ideal for debugging the question why certain index or plan is not executed.
- Returns [QueryPlanner](https://www.mongodb.com/docs/v6.0/reference/explain-results/#queryplanner) and [ExecutionStats](https://www.mongodb.com/docs/v6.0/reference/explain-results/#executionstats) objects


# Index

- To get the list of indexes. Call `db.collection.getIndexes()` method.
- To get the number of memory required by indexes in bytes, run `db.collection.stats().indexSizes` 
- Like all databases, indices are changed when document is inserted or removed. But it's only changed when indexed field changes.

## Limitations of Index
- Can have max 64 index in collection.
- Write performance usually degrade at 20-30th index.


## Single Field Index
- Creating index for one field
- Specified as field name and direction
	- But direction doesn't matter here
- Indexing an object type does not index all the values separately
	- Single index on object type is only useful when we are matching entire object, deeply, as it is.

### Index Options
- Unique Index: 
	- Enforces Uniqueness in collection
	- `db.collection.createIndex({field:1},{unique:true})` 
	- `NULL` is a valid value and thus only one document is allowed with null value,
- Sparse Index
	- Only contain entries which have the given field present in the document, even if the value is null.
	- `db.collection.createIndex({field:1},{sparce:true})`
	- [[#Spherical Geometry(`2dsphere`)|`2dsphere`]],[[#Cartesian (`2d`)|`2d`]],`geoHayStack` and `text` are always sparse by default
- Partial Index
	- Superset of Sparse Index
	- Only index documents that meet specified filter expression.
	- `db.createIndex({field:1},{partialFilterExpresstion:{expression}})`
	- Can greatly reduce index size.
	- Useful if that expression is often used in find or update queries.

### Hashed Indexes
- 20 byte md5 of BSON value
- Support exact match
- Can't be unique
- Reduce size if the values are larger than 20 bytes
- Stored randomly so searching hashed values in BTree can be more expensive.
- `db.collection.createIndex({field:"hashed"})`

### Multi Key Index
- Index on an array.
- Just a [[#Single Field Index.|single field index]] on array field.
- An index entry created on each value of array.
- Can support primitive, documents and sub documents

## Compound Index
- Index with more than one fields.
- Unlike [[#Single Field Index|Single Field index]] Field order and their sort direction matters.
- Limit: Upto 32 fields per index.
- Can create like single field index but with more fields specified.
- Index will be used as long as the first field mentioned in index are in query.
- Index is fully utilised when first N fields are used in Query. (Called Index Prefixing)
	- e.g. For a multi key index `{a:1,b:1,c:1}` 
	- If we use only `a` in query, index will be utilised and scan all documents with `a`. Which is optimal and intended
	- If we use `a,b` or `a,b,c` index will be utilised and scan all documents with `a,b` or `a,b,c`, this is also optimal.
	- If we use `a,c` Index will be utilised and scan all documents with `a` (this is intended stage), then scan all documents with `b` (which was never desired or even mentioned in query) and then filter out `c`, thus despite the index being used it is not utilised fully.

### ESR Rule
- While creating compound index consider following rule. #recommendation 
	- Equality before Range
	- Equality before Sort
	- Sort before Range
- When putting Equality first, our index scan is limited to only those document whose value we want.
- When putting Sort before range, we will avoid expensive sorting computation, which 
> [!tip] When creating compound index with [[#Multi Key Index]] only one key with Array is allowed, as more than one key with array will result in all possible combinations being stored which is not optimal.

## Index covered queries

- If we do a query and a projection where all the fields covered by index, we don't have to go to the document, this is called Index covered queries.
- Gotchas
	- Usually index doesn't cover `_id` field, so we have to remove it from projection
	- When we involve array fields, we can't use index covered queries, because
		1. Indices are not stored as arrays, (not sure how? but) while looking at indices, we don't know if it's a regular single field index or multi key index.
		2. For getting all data in array, we already need to go to document because this is not covered in index.
	- Index are not BSON values. Conversion is applied.

# Geospatial Index
- Most index fields are ordial.
>[!info] Ordial field means they can be arranged in order, where it's easy and clear that `A>B>C`. Thus can be sorted and searched efficiently.

- Coordinates are not Ordial, they are pair of doubles, thus we can sort them by EAST>WEST or NORTH>SOUTH. But can't combine both sorts.
- That's why Geospatial Indices are ==special and non ordial==.
- To index Geopatial data, database solutions either use `Geohashes` or `ZTrees`. MongoDB uses `Geohashes`.
- All types of Geospatial indices are sortable by distance.

- Used to determine geometric relationships
	- All shapes with certain distance of other shape.
	- Whether or not shape falls within other shape.
	- If two shapes intersect.
- Used to find all maps type use-cases.

## Types

### Cartesian (`2d`)
- Simple lat long pair
- Don't take earth's curvature into account
- Good for small distances
- `db.collection.createIndex({field:"2d"})`

### Spherical Geometry(`2dsphere`)
- Full set of GeoJson objects
- Useful for larger areas
- `db.collection.createIndex({field:"2dsphere"})`


# TTL Index.
- Similar to Redis TTL.
- It's not a special index, it's a flag on an ==index on a date==
- MongoDB removes the document using the index based on time
	- Server runs a job periodically in background to remove TTL index.
- `db.createIndex({fieldWithDateDataType:1},{expireAfterSeconds})`
- Alternatives to use TTL index.
	- Set ExpireTime in document, mentioning the time when document should be deleted.
	- Create a TTL index in ExpireTime with `expireAfterSeconds` set to 0
- Has some unknown write load.

# Text Index
- Superseded by Atlas Search
- Useful for performing search when MongoDB is not hosted in Atlas.
- Indexes tokens used in string fields.
	- Allows to search for `contains` type search.
- Algorithm
	- Split text fields into words
	- Ignore `stop words`:(Common non useful words like `a`, `an`, `the` etc)
	- Apply stemming: (`running`, `ran`, `runs`, `runner` are mapped to `run`).
	- Take the stemmed words and make a [[#Multi Key Index]].
- Queries are `OR` by default
- Supports majority western languages
- Can be compound index.
- `db.collection.createIndex({field:"text"})`
- [`$meta`](https://www.mongodb.com/docs/manual/reference/operator/aggregation/meta/#mongodb-expression-exp.-meta) useful for having additional information like `textScore` or 

## Limits
- No AND
- No Fuzzy matching
- No Intersection
- No Wildcard Entries
- Many index entries per document, thus making slower writes.



# Wildcard Indexes
- Index all fields from root or a subtree.
- `db.createIndex({'$**':1})` creates wildcard index from root
- `db.createIndex({'field.$**':1})` creates wildcard index from subdocument `field`.
- To include specific fields use `{wildcardProjection:{includingFields:1}}` or `{wildcardProjection:{excludingFields:0}}` when creating index.
- Slower writes 
- When being used, it creates all single field indexes from a wildcard index in memory and use that in the plan.

# How MongoDB selects indexes. 
 ^plan-cache
- MongoDB uses `planCache`.
	- Where it remembers query shape (Query without values) and index used against that query.
	- If `planCache` has that query stored, MongoDB uses that index to execute the query.
	- If `planCache` doesn't have the query. It does following
		- Picks all candidate indexes.
			- We can hide index from query planner using `hideIndex()` method.
		- Runs query against them and see which one is best.
		- Add's the selected (or best) index in `planCache`.
	- Can query `planCache` using `db.collection.getPlanCache()`
	- Eviction of `planCache`.
		- Using that index becomes less efficient
		- A new relevant index is being added in that collection
		- The server is restarted.
- If someone wants to use the index forcefully, [`hint` method](https://www.mongodb.com/docs/manual/reference/method/cursor.hint/) should be used.
- Index can be used for Regex...Only if  ^regexIndex
	- If anchored at start (Like : `{name:/^Hon/}`)
	- Is case sensitive
	- Mostly a range query.
- We can define [collation](https://www.mongodb.com/docs/manual/reference/collation/).

## Index in production

- Creating index in large data can be problematic. As it needs to scan a large dataset to build index in foreground, thus blocking reads.
- Creating index in background may result in index not applied in time.
- There were two types of index `foreground` and `background` indexes pre 4.2.
- Since 4.2 these values are ignored and indexes are being built hybrid, thus having best of foreground and background builds.

## Hidden Indexes
- Prevent database to include the given index in plan.
- No longer used in `find` or `update`
- Still updated and can be re-enabled.
- Unique constraints are still applied
- TTL Index will still be removed.
- Can be used to test if it's ok to drop an index.


# Finding slow operations

- Most common source of slow operation is missing index.
- Second most source is bad client code.
- Two ways to find inefficient queries
	- DB Log
	- [[Mongod#Profiler|Profiler]]
		- Profiling is not available in Atlas Shared instances.

- DB Log ^dbLogs
	- `db.adminCommand({getLog:'global'})`
		- Gets last 1024 entries
		- Currently they are JSON but before 4.4 they were plaintext. #version #gotchas
	
		- Options for `getLog`
			- `global`: Combined output of all recent entries (Something we're interested in)
			- `startupWarnings`: Return entries that may contain errors or warnings when current process started.
			- `*`: Returns a list of available values of getLog Command
	- Another way is to get the logs if we manage MongoDB via self managed, atlas or ops manager.

- Slow Query are those which takes longer then [[Mongod#^slowMs|slowMs]] which is default to 100ms and can be configured.
- If we turn on [[Mongod#Profiler|profiler]] we can record slow operations or all operations in [[Mongod#^systemProfileCollection|`system.profile` collection]]
	- However it causes twice the writes and one write and read for slow operations and thus it's desirable to either turn off the profiler or using logs if we are using MongoDB 4.4 or newer. #version #gotchas 


## Causes of Slow Operations
- Anything that leads to disk reads, like
	- Missing index
	- Too large document 
	- Wrongly or under-utilised index (See [[#ESR Rule]])
- If we are writing more than the memory that can be easily flushed to disk, it may temporarily create slow writes.
- Locks.
	- Larger locks may caused by renaming Databases or collections.
- Excessive CPU usage.
	- Constantly logging in and out
	- Document Lock contention ([[Intro to Mongodb#MVCC|See MVCC]]).
	- Inappropriately large arrays 
	- Running code in db.
- Too complex Nested Joins using `$lookup` or `$graphLookup` 

# Aggregation

- Some stages, if used in start of the pipeline, are transformed into `find()` by Query optimiser.

| Aggregation stage | Equivalent query operator for optimising |
| ----------------- | --------------------- |
| `$match` | `.find()` |
| `$project` | `.find({},{PROJECTION}`) |
| `$sort` | `.find().sort()` |
| `$limit` | `.find().limit()` |
| `$skip` | `.find().skip()` |
| `$count` | `.find().count()` |

- [Aggregation Pipeline Stages](https://www.mongodb.com/docs/manual/reference/operator/aggregation-pipeline/)
- [Aggregation Pipeline Operators](https://www.mongodb.com/docs/manual/reference/operator/aggregation/)


- Expression variables in aggregation pipelines are denoted with `$$`
- [Aggregation Variables](https://www.mongodb.com/docs/v6.0/reference/aggregation-variables/)

## Optimising aggregation
- Some common tips
	- Start with step that filter out most of the data.
	- Two consecutive matches should be combined with `$and` clause
	- Consider project as last step in pipeline. Pipeline can work with minimal data needed for result. #recommendation 
		- If we do project early in the pipeline chances are that pipeline may not have some variable that it needs.
		- If we need to have a calculated result `$set` (equivalent of `$addField`) stage is recommended. #recommendation 
	- `$unwind`ing an array only to `$group` them using same field is anti-pattern.
		- Better to use accumulators on array.
	- Prefer streaming stages in most of the pipelines, use blocking stages at the end.
		- `$sort(withoutIndex)`,`$group`,`$bucket`,`$facet` are blocking stages.
		- Streaming stage will process documents as they come and send to the other side.
		- Blocking stages wait for all document to enter in the stage to start processing.


## Aggregation in Sharded cluster

- On [[Sharding|sharded clusters]], operations run in parallel where possible.
- Combining operations run in different location
	- An easy merge operation which require little to no calculation run on [[Sharding#`mongos`|`mongos`]]
	- Other calculations can happen on primary shard or random shard depending on MongoDB server version and operation.

