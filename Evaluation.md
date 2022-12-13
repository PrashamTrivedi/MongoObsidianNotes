# Java App (Mongomart)

## Update Java MongoDB Driver (Priority 1)

Client is using MongoDB java driver 3.5.0, and their MongoDB server is running `MongoDB 4.2.23 Enterprise` version. 

Java Driver 3.5.0 supports till MongoDB version 3.4. Source: https://www.mongodb.com/docs/drivers/java/sync/current/compatibility/. To access all the features of MongoDB 4.2, Java driver version 3.11 and above are required, our recommendation is to use Java Driver version 4.2 or above. 

You can see changelog [here](https://github.com/mongodb/mongo-java-driver/release), which contains 

## Reuse Connection and database variables. (Priority 2)

Each operation contains this piece of code. 

```java
MongoClient mongoClient = new MongoClient(new MongoClientURI(mongoURIString));

MongoDatabase martDatabase = mongoClient.getDatabase("mongomart");
```

This code will try to create a new connection with mongodb, this takes more time to connect and more memory is used.

Instead of repeating it, it should be re-used.

## Drop Sort by `_id` (Priority 2)

Customer is sorting by `_id` value in listing in `ItemDao.java:131`. In `ItemDao.java:151`, sorting by `_id` is sorted descending and code from `120 to 122` iterates through reverse order which is computational overhead.

All the collections are sorted by `_id` by default. 

## Drop `category` in Cart. (Priority 3)

Customer is adding `category` field in cart which isn't used anyway.

## Limit number of reviews in `item` document (Priority 1)

Client is directly adding reviews in item document. There is no limit on how many reviews are stored in database.

This can lead to performance penalty later on. Unbound arrays will increase document size significantly and make other operations slower.

## Embed Address in Invoices (Priority 1)

All Invoices and address are stored in separate collections. While querying the invoices, address is queried in `address` collection via `_id`. 

This will require additional computation, which is costly operation, requires more network roundtrips. 

Embedding `address` will reduce latency and will return the documents faster.

## Store Items as number in Invoices (Priority 1)

All Invoices store `items`, which is Array of numbers, but UI only cares about number of Items, i.e. size of `items` field. 

Storing an array is less optimal when you only require the number of items stored in it and not the actual content. 

Removing `items` as an array and storing count will be lot of beneficial for retrieval.

## Use `itemCollection.count()` in `textSearchCount` (Priority 1)

When doing `textSearchCount()` client uses `find` query to get the data, projects only `_id` , stores into ArrayList and then returns the `size` of the ArrayList.

This is un-optimal, specially when `itemCollection.count()` returns the number of documents without doing such calls.

## Remove `limit` Options from `$geoNear` Queries (Priority 2)

When searching `buildClosestToLocationPipeline`, `limit` is appended in `$geoNear` queries.

Starting with MongoDB v 4.2, `limit` is no longer working with `$geoNear`. To mitigate it, another stage with `$limit` is used.

You should remove `limit` from `$geoNear`.

## Use `projection` in StoreDao (Priority 2)

When Querying ZipCode or City and State, we observed that in `find` operation, whole document is queried only to fetch `lat` and `long`. 

Fetching whole document will require more fields in memory and bigger object in network travel.

It is advised to use Projection in StoreDao.

