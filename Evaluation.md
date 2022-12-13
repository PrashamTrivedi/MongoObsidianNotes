# Java App (Mongomart)

## Reuse Connection and database variables. Priority 1

Each operation contains this piece of code. 

```java
MongoClient mongoClient = new MongoClient(new MongoClientURI(mongoURIString));

MongoDatabase martDatabase = mongoClient.getDatabase("mongomart");
```

This code will try to create a new connection with mongodb, this takes more time to connect and more memory is used.

Instead of repeating it, it should be re-used.

## Drop Sort by `_id` (Priority 2)

Customer is sorting by `_id` value in listing.

All the collections are sorted by `_id` by default. 

## Drop `category` in Cart. (Priority 1/2)

Customer is adding `category` field in cart 