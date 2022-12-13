# Java App (Mongomart)

## Reuse Connection and database variables.

Each operation contains this piece of code. 

```java
MongoClient mongoClient = new MongoClient(new MongoClientURI(mongoURIString));

MongoDatabase martDatabase = mongoClient.getDatabase("mongomart");
```

Instead of repeating it, it should be re-used.


