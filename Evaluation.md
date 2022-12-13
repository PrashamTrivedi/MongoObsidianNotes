# Java App (Mongomart)

## Update Java MongoDB Driver (Priority 1)

Client is using MongoDB java driver 3.5.0, and their MongoDB server is running `MongoDB 4.2.23 Enterprise` version. 

Java Driver 3.5.0 supports till MongoDB version 3.4. Source: https://www.mongodb.com/docs/drivers/java/sync/current/compatibility/. To access all the features of MongoDB 4.2, Java driver version 3.11 and above are required, our recommendation is to use Java Driver version 4.2 or above. 

You can see changelog [here](https://github.com/mongodb/mongo-java-driver/release), which contains 

## Reuse Connection and database variables. Priority 1

Each operation contains this piece of code. 

```java
MongoClient mongoClient = new MongoClient(new MongoClientURI(mongoURIString));

MongoDatabase martDatabase = mongoClient.getDatabase("mongomart");
```

This code will try to create a new connection with mongodb, this takes more time to connect and more memory is used.

Instead of repeating it, it should be re-used.

## Drop Sort by `_id` (Priority 2)

Customer is sorting by `_id` value in listing in `ItemDao.java:131`

All the collections are sorted by `_id` by default. 

## Drop `category` in Cart. (Priority 1/2)

Customer is adding `category` field in cart which isn't used anyway.

## Limit number or reviews in `item` document (Priority 1)

Client is directly adding reviews in item document. There is no limit on how many reviews are stored in database.

This can lead to performance penalty later on. Unbound arrays will increase document size significantly and make other operations slower.

## Embed Address in Invoices (Priority 1)

All Invoices and address are stored in separate collections. While querying the invoices, address is queried in `address` collection via `_id`. 

This will require additional computation, which is costly operation, requires more network roundtrips. 

Embedding `address` will reduce latency and will return the documents faster.

## Remove Items from Invoices (Priority 2)

