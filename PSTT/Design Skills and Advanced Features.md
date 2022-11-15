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

- Like triggers in SQL database
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


# Developer Internals

## BSON Data types
- Reiterating earlier....![[Intro to Mongodb#^bson]]
- Use proper type. 
	- Using number enables performant range queries #recommendation 
- Default number type in mongo shell is `double` #gotchas 
- Unlike Javascript MongoDB has native date type
- Special Type
	- `MinKey`: Smallest value
	- `MaxKey`: Largest Value
	- `TimeStamp`: Used for internal use, like storing [[Replication#oplog.rs|Oplog]] entries.

## Null Handling

- In MongoDB missing field is same as null.
- Projection in non existent field will return null.
- If we query for null, it will return `null` as a value [^3] and missing field.
- In aggregation, `$lookup` and `$graphLookup` will match null values from source to null values in destination.

## Collation and sort ordering

[collation in MongoDB](https://www.mongodb.com/docs/manual/reference/collation/)

- All text is stored in Unicode - as `utf8`.
- Default Sort order is unicode code point order. 
	- Good for english but may not be good for other languages
	- Each languages can have different ordering of arranging words and different things 
- Collation specify language specific rules.
- With that..
	- Sorting order can be changed
	- If case is considered or ignored
	- If to match diacritics e.g. Jose = Josè?
- For same field with different values, sorting will be
	- `null<numbers<strings<objects` #gotchas 

## Type Bracketing

- When comparison MongoDB auto converts data types
- Searching for 5 will find integer, long and double. But not string
- Index stores the number mainly (along with type but not for finding and sorting purpose).
- In `find` type if queries `{x:{$lt:5}}` null is not < 5
- In `aggregation` type of queries `{$lt:["$x",5]}` null is < 5

## Object Sorting

- When objects are compared, field order matters
- Objects are compared field by field
- If names differ order is by first changed field in order they are inserted
- values differ, order is by value

## Locking

- Single Document writes are atomic
- Write operation takes locks for writes only reads can happen regardless. ![[Intro to Mongodb#MVCC|See MVCC]]



[^3]: Having field with value null.


# Developer Good Practices

## Idempotent Operations
- Idempotent: If an operation is idempotent, it will output same result no matter how many times it's applied. ^idempotency
- Doing same operations again should not result in duplicates
- Following write operations are idempotent
	- Insert with supplied `_id` values. (You can't insert same id twice)
	- Delete (You can't delete same thing twice)
	- Update (If we're explicitly updating values and not using relative updates like increment or decrement, updating same values will be ignored.) 
- If there is an non idempotent operation (Usually an update with relative values or insert without `_id`) use condition to identify duplicate records or update only records which are not updated.

## Authentication

- It's complicated for security reason
	- Good practice to have a singleton object that authenticates...
	- ...And have a connection pool of authenticated client connections #recommendation 

## ORM/ODM (Object Relation/Document Mapper)

- Third party libraries which maps native language object with database native schema.
- Like Spring Data, Morphia, Mongoose
- ORM/ODM is an idea borrowed from Relational Database where objects/JSONs/maps have no one to one mapping with columns, but should have.
>[!warning] 
>
> - No ORM/ODM is officially supported by MongoDB
> - They are not necessarily follow best practice
> - They are not always using latest drivers and thus missing out capabilities of latest versions
- Instead of using ORM/ODM, it's better to use Data Access Layer in the apps which are responsible for managing all the data. #recommendation 

### CODECs
- [CODEC Document](https://www.mongodb.com/docs/drivers/java/sync/current/fundamentals/data-formats/codecs/)
- Many MongoDB drivers (like Java) support CODECs
- They are Classes registered with drivers to convert native Classes to and from BSON.
- Default POJO/POCO (Plain Old Java/C# Object) allow have 1:1 mapping between class instances and underlying MongoDB documents
- Custom CODECs allow developers to define custom mapping.
- They are not Flexible as Data Access Layers, but provide first party support to ODM concept in some languages.

# Schema Design

## BSON
- BSON is native format for Data
- Drivers convert BSON to native objects (Maps/Dicts/JSONs etc)
- Basically a list of named and typed values with length stored optionally
- When scanning for values BSON tries to scan all fields one by one. And skips one that is not required in Search
- Advantage of storing embedded documents using this point in mind is it can skip multiple fields together.
	- E.g. When we have `address_locality`,`address_sublocality`,`address_state`,`address_country` in a flat document we always need to scan 4 fields even though we don't want any address fields.
		- If we use `address:{locality,sublocality,state,country}` we can skip entire document, thus saving memory 
### Containers: AKA Arrays and Embedded Documents
- [BSON Spec](https://bsonspec.org/spec.html) lists Embedded document and array both stored as documents, it's just a byte separating them. #gotchas 
- Just like Large and Flat documents, large arrays are also bad for performance.
- Keep max 200 elements in array or max 200 fields in a document #recommendation 
- Use `bsonsize` function in mongo shell to see the BSON size of an object #gotchas 
- Use arrays when
	- You need to `$push` to the end (i.e. Appending item regularly)
	- You don't have unique identifiers
	- Sparse data is not issue (Nulls in elements are tolerated)
	- Values needed to be indexed.
- When objects are used
	- Projections are faster than `$filter`
	- Querying single member is faster.

## RDBMS vs Document DB
- RDBMS are designed based on shape of data, not how they are intended to use.
	- Data stored on different places based on their property
	- Mapping is often required
	- Not always a faster and straight forward retrievals 
	- Often require Transaction to update related data together
- Document DBs should be designed based on how they are intended to use.
	- Data should be stored in same document according to usage patterns.
	- Mapping is not required, embedded document is often used to denote some relationship.
	- Designed to be faster and straightforward retrievals.
	- Single atomic update to update related data.

## To Link or To Embed

### Terminology
- Parent document: The document which is being queried to satisfy the operation.
	- E.g. Order, Movie, Football Club.
- Child document: The document which is not main part of the operation, but linked with parent document to provide extra information, it may have own lifecycle different to parent.
	- E.g. Address In the Order, Actor in the movie, Player or manager in the football club.

- Embed: Have a subdocument in main document
- Link: Have an id in main document which points to another document.

### Considerations #recommendation 
- Rule of thumb: If a parent document should not be updated when child document is updated, consider embedding #recommendation 
	- E.g. Address in Order, should always be embedded to know the historical data.
- If parent document must be updated when child updates itself, consider linking.
	- E.g. An actor changing their name should be updated in movies.
		- Like Ellen Page changed to Elliot Page is reflected in all their movies
- If Parent Document only needs some information about child which is rarely changed consider partial embedding with `_id` of child to query other data.
	- E.g. A players' name, height, national team etc does not change even if they change clubs, places or even appearances. In such case the unchanging information can be embedded, also we should have `_id` of the player in embedded document so that we can link or aggregate all data of the player. ^subsetPattern
- If searching within the children are essential, consider embedding.
- Even if embedding, consider adding `_id` field of child to access whole document of child. #recommendation 

## Payload vs Process fields
- Payload fields: Fields where values are there to storage or retrieval
	- Just there for storage and retrieval, not playing any part on workflow.
	- E.g. About field of a profile.
- Process fields: Metadata we examine in Database 
	- The fields who are used in querying, aggregating or deriving other data.
	- E.g. Address of profile, used to get nearby users or DOB from which age is calculated to provide age appropriate content.


## Dynamic Models

### Evolutionary Dynamic Schema

- Schema changes as App changes
- Schema versioning
- No need to immediate conversion/migration

### Payload Driven Dynamic Schema

- The app's purpose is to store some arbitrary data, developer has no control over it.
- Hard to index and hard to make performant
- Unpredictable schema


### Data Driven Dynamic Schema

- Field names are actual values which are mapped to other values
- Requires a new development approach where developer don't know the key beforehand and have to traverse through all keys.
	- Equivalent to querying with `$exists` instead of `$eq`.
	- Highly performant, can save around 30% of data if not more.

## Design patterns

- Not universally understood and accepted 

### Attribute pattern
- Follows [[#Data Driven Dynamic Schema]]
- Not all values may present in record
- Easy to search, instead of querying value where name is X, we query the value of X
- Indexing is quite difficult.
- Confusion over [[#Payload vs Process fields]]
- Use either [[Optimizing Storage#Wildcard Indexes|Wildcard Index]] or Embedded document with a common name.
	- Like attributes for E-commerce.

### Bucket pattern
- Most common design pattern
- Main document has an information of their bucket (e.g. a parent entity where the data belongs)
	- E.g. IoT data consists sensor name
	- Bank transaction data consists account number
- Store many small, related data items.
- Speeds up retrieval.
- Indices are smaller 
- Enables Computed Pattern

### Computed Pattern

- Never recompute if you can pre-compute.
- Used when computed reads are more often then computed writes.
	- E.g. Live score (Esp in slow-er sports like football) where there are limited process to write, but unlimited readers.
- Sometimes we may require to update some additional summary records too


### Versioning Pattern
- Good example of payload vs processing.
	- For current version it's processing, for previous versions is mostly payload

### The Subset pattern
- Using link to other record
- Maintain a subset of whole child data embedded for speed.
- Example![[#^subsetPattern]]
### Outlier or overspill pattern.

- Like [[#Bucket pattern]], a new record if too many items
- There is a main record and another flag to say if there is an overspill situation.
- Like IMDB, movie information embeds main cast and crew, and links other cast and crew (a bigger information) in separate document.


