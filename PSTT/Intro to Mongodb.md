---
aliases: DF100
---
- Two things that are hard to do in distributed systems
	- **Joins**: Database level joins are difficult when data we are joining is residing in different machines.
	- **Unreliable Network Communications:** Unless a machine confirms or acknowledges the message, one machine can not know if the other machine is alive, processed all the messages properly or not.
		- Transactions are causality of this problem

- ==BSON== : ==B==inary ==S==erialized ==O==bject ==N==otation

- [BSON Data types](https://www.mongodb.com/docs/manual/reference/bson-types/) ^bson

- [Object ID](https://www.mongodb.com/docs/manual/reference/bson-types/#objectid)
	- 12 bytes value
		- 4 byte timestamp: representing the ObjectId's creation, measured in seconds since the Unix epoch.
		- 5 byte random value generated once per process or per machine (Likely `mongod`)
		- 3 byte incrementing counter initialised to a random value

## Insert Many and Orders
- `insertMany` is `ordered` by default, means all collections are inserted in order they are passed, and the operation stops when an error is encountered. And no documents after the document generating error will be inserted. 
	- To fix this use `{ordered:false}` thus, instead of `db.collection.insertMany(array)` we should use `db.collection.insertMany(array,{ordered:false})` 
	- This is called **Unordered Inserts**.
- Ordered insert is useful when you explicitly want to stop on error.
- Unordered inserts have two advantages 
	- It inserts all document that does not have any error.
	- In sharded clusters, it enables to insert into shards in parallel.

## Cursor
- When we run a query, a `cursor` is returned and by default it prints the result of first 20 documents according to `find` parameters.
- If we assign a cursor to a variable, that variable doesn't do anything, that cursor variable should call some method to retrieve the data from server like `hasNext`, `next`, `itcount` or `forEach` where it connects the server, retrieves some data and prints the result. 
- Some methods available in `mongosh` for cursor are listed [here](https://www.mongodb.com/docs/manual/reference/method/js-cursor/) ^cursorCache
	- Cursor fetches the data from the server in the batches and caches them in memory, and during `hasNext` or `next` calls, cursor tries to fetch the data from cache and goes to server if data is not present in cache. ^8aaf23
	- Default batch size in shell during initial call is 101 documents or 1MB data (Whichever is larger). 
	- When exhaust initial batch is exhausted, server will call `getMore` to receive next batch, which fetches in 4MB chunks. ^cursorCacheGetMore

## Querying

- When using comparison operators, syntax should be `{fieldName:{$OP:value}}`.
- When using logic operators. `$and`, `$or` and `$nor` follows this syntax  `{$OP:[{field:value},{field:value}]}`.
	- When using `$not`. Syntax should be `{$not:{field:value}}`
- When using aggregation operators or expressions, syntax should be `{$OP:["$field",value]}`
- `$elemMatch`: The array operator which can be used in query and projection. 
	- If used in query, it only includes those documents where at-least one element in given array matches the given condition.
	- If used in projection, it only shows those elements in given array that satisfies the given condition.

## MongoDB Upsert

In MongoDB, Update Operations are done with following syntax `db.collection.update({Query},{UpdateOperation})`. By default it will only update the matched document. But there is another clause which allows upsert, if in update syntax we add `{upsert:true}` as third argument, it will insert the data noted by `UpdateOperation` if there are no documents matched by `Query` criteria.

### Update ArrayFilters

^e767f5

- [From Documentation](https://www.mongodb.com/docs/v4.4/reference/method/db.collection.updateMany/#parameters): An array of filter documents that determine which array elements to modify for an update operation on an array field. 
- My Understanding:  ^a7681b
	- Array Filters are part of query documents which determine exactly which members of array should be updated.
	- Array filters comes after `query` and `update` predicate.
	- In `update` predicate, arrayFilters are used as {`arrayField.${filterName}`:update}
	- Format:
		- For Scalar arrays: `{arrayFilters:[{filterName:{queryPredicate}}]}`
		- For Object arrays: `{arrayFilters:[{filterName.property:x,filterName.property:y}]}`
			- i.e.: Instead of having an explicit filter name, we use the filter name in query predicate, all the members match `arrayFilter` condition will be stored in `filterName`

## MVCC

Stands for ==M==ulti==V==ersion ==C==oncurrency ==C==ontrol.
Updates on document is done via the following way.

- Each data has a version set at specific number at a time
- A Copy of the data is created.
- Update happens on that copy. 
- Once an update is done, the data's version is increased.
- All the reads are done on the latest version.
	- i.e. During the updates, reads happen on original data which has the latest version.
	- After updates reads happen on new version
- If another process tries to update on same data, it tries to copy the data before the original update happens, update it and increase the version number.
- At the time of update, the process find out that a data with same or increased version already exists, so that process retries the update.