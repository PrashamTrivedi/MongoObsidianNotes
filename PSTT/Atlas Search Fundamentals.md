## Search Index Parameters

- Index Analyser
- Search Analyser
- Dynamic Mapping
- Field Mappings

- In absence of other parameters, scoring will be done based on what % of the document is the search term.
	- E.g. If searched by one word, shortest document will be returned.


## Type Mapping

| BSON Type | Included in Dynamic Mapping | Atlas Search Type |
| --- | --- | --- |
| Double | Yes | number |
| 32 bit integer | Yes | number |
| 64 bit integer | Yes | number |
| String | Yes | string | 
| Date | Yes | date |
| Object | Yes | document |
| Array | Yes | type of elements |
| ObjectId | No | objectid |
| Boolean | No | boolean |
