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


- [Atlas search docs](https://www.mongodb.com/docs/atlas/atlas-search/)
- [Atlas Analyzers](https://www.mongodb.com/docs/atlas/atlas-search/analyzers/)

| Analyzer | Description |
| --- | --- |
| Standard | Uses the default analyzer for all Atlas Search indexes and queries. |
| Simple | Divides text into searchable terms wherever it finds a non-letter character. |
| Whitespace | Divides text into searchable terms wherever it finds a whitespace character. | 
| Keyword | Keeps the text as it is |
| Language | Provides a set of language-specific text analyzers. |

- [Custom Analyzers](https://www.mongodb.com/docs/atlas/atlas-search/analyzers/custom/)
- 