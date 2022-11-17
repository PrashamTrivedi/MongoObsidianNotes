## Search Index Parameters

- Index Analyser
- Search Analyser
- Dynamic Mapping
- Field Mappings

- In absence of other parameters, scoring will be done based on what % of the document is the search term.
	- E.g. If searched by one word, shortest document will be returned.


## Type Mapping

| BSON Type | Included in Dynamic Mapping[^2] | Atlas Search Type |
| --- | --- | --- |
| Double | Yes | number |
| 32 bit integer | Yes | number |
| 64 bit integer | Yes | number |
| String | Yes | string | 
| Date | Yes | date |
| Object | Yes | document |
| Array | Yes | type of elements |
| ObjectId | No[^1] | objectid |
| Boolean | No [^1]| boolean |

[^1]: Object Id and Booleans are not included in Dynamic mapping because usually you don't do any search or range, you are doing equality on them.
[^2]: Not included in dynamic mapping does not mean it can't be indexed at all.


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
- [Operators and collectors](https://www.mongodb.com/docs/atlas/atlas-search/operators-and-collectors/)


## Combining Operators

Below operators must be included in [`compound`](https://www.mongodb.com/docs/atlas/atlas-search/compound/#std-label-compound-ref) operator.

- `must`: All Clauses have to match, ==equivalent to AND==
- `mustNot`: None of the clauses have to match, ==equivalent to NOT AND==
- `should`: N terms have to match, N defaults to 0, if something matches the score increases.
- `filter`: Same as must but not affected by score.

