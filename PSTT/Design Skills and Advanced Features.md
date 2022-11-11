# Beyond Storage

## Regex
- Useful for LIKE/CONTAINS kind of query, patterns and wildcard matches.
- Regex limitation w.r.t. index. ![[Optimizing Storage#^regexIndex]]
- Regex is also a BSON type
	- Storing Regex is also unusual.
	- Recommendation: Create them using native regex in language. #recommendation

## Schema Validation

- By default there is no enforcement of schema. 
- Some restrictions like
	- It must have an `_id`
	- Unique index constraints apply
	- Must be <16 MB
	- Duplicate fields should not be in document #recommendation [^1]
- We can add additional constraints like...
	- Some field must have some type.
	- Some field should not have more than given length
- We can also specify a query as validators
- We can specify `$jsonSchema` as custom validator #documentTodo
- Every insert or update validates input against the validators and rejects as errors or prints warning in log.
- Documents inserted before validations are created won't be validated.
- Prior to 5.0, validator only said validation failed, from 5.0 and onwards validator says why it is failed. #version #gotchas


[^1]:  In drivers, developers won't see duplicates anyway, the entries unmarshalled last will be in the document.

## GridFS

- Specs and driver API to store Larger files in MongoDB
- CMD: `mongofiles`
- Splits file to multiple documents to avoid the limit
- Can fetch whole file or byte range
- For smaller files it's easier to user Binary Field
- Alternative to Block Storage 
- Consider Block Storage like S3 because they are cheaper #recommendation 

## Change Streams

- Listen for writes to a Document, Collection, Database, Replicaset, Shards or Instance
	- Can filter what kind of notification we want to receive
	- Can receive whole or delta
	- Stop and resume notifications at will
		- Server sends resume token, if client passes resume token and it isn't expire, client gets changes after resume token.

- Notification is only sent when changes are written to majority of nodes.

- Drivers handle this differently
	- Block until an events is received or timed out
	- Asks if there are any changes or not in the stream
	- Provide a callback function
	- etc..


## Retryable reads and writes

- Mechanism to avoid handling errors in client side.

### Retryable writes:
- Automatically retry writes in event of HA failover
- Default for 4.2+ drivers and Atlas connection strings #version #gotchas 
- Checks..
	- Cluster is available
	- And primary does not already seen this

### Retryable reads:
- Automatically retry reads.
- Does not apply on [[Intro to Mongodb#^cursurCacheGetMore|`getMore`]] operations 
	- Because if server gets down, the cursor and it's initial batch doesn't exist.

## Multi Document Transactions

- Transaction[^2] that writes to more then one documents.
- Snapshot level isolation.
- Requires atleast a replicaset #documentTodo 
- Creates write contention.
	- Other writes that happen at same time as the transaction has to wait or face errors.
- Use proper schema design in place of using transaction #recommendation 


[^2]: Steps: Begin>Do A>Do B>Do C>Commit If succeeded>Rollback if error


## Bulk Write Operations
- Group together multiple writes in a single network call.
	- `insertMany` is a bulk.
	- `updateMany` is single operation
	- Bulk Write API sends many changes in single command #documentTodo 
- Avoids latency costs and network round trips #recommendation 
- Can be used inside transaction
- Can be ordered or [[Intro to Mongodb#Insert Many and Orders|unordered]].
> [!caution] Bulk write only works in single collection


## Javascript inside MongoDB

- MongoDB has a javascript engine in server.
	- Ideally this should be disabled
	- Slow and there to support legacy code
	- Use native aggregation instead #recommendation 
- Javascript `$function` aggregation in 4.4 #version #recommendation 
	- Step on roadmap to deprecate `mapReduce`.
	- Not a general purpose aggregation task
	- Only use when current aggregation pipeline does not support it
	- Don't use `eval`, `$where` or `mapReduce`.

## Atlas Search

- MongoDB's native text search capability via `Text Index` 
![[Optimizing Storage#Text Index]]
- Atlas is more powerful compared to Text Index
	- Uses Lucene 
	- Has own indexing process
	- More powerful and capable compared to Text Index.

## Atlas Triggers

- Not to be confused with [[#Change Streams]]
- Not native feature of Database
	- Built using Realm Backend as a service
	- For Hosted MongoDB only
	- Written in JavaScript
	- Can be fired on data changes or as cron.

## Views

- Concept from RDBMS
- View Only Collections
- Defined by aggregation pipeline
- Hard to Index
- Have own security so can be use to redact main collection and only allow to see View.

