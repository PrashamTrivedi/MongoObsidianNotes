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


## Autocompletes

- [Autocomplete Docs](https://www.mongodb.com/docs/atlas/atlas-search/autocomplete/) 
- When string is broken down into tokens for autocomplete, we use one of two methods.
	- **nGram**: All sets of consecutive letters which ==can appear anywhere in the string, also including whitespace==. 
	- **egdeGram**: Always search at start of the word.

### Tips for using Autocomplete

- ==Consider index size, it can grow very large==
- `nGrams` create larger index then `edgeGrams`
- The smaller the gram size, larger the index

## Highlight

- Use `highlight:{path:}` to highlight the relevant information
- And in projection use `$meta: 'searchHighlights'` in your desired field to show highlights.
- It shows what matched, not what was queried
- Adds latency
- Does not work with autocomplete index
- By default lucene stores original data to highlight, so if highlight is not used, remove `store` option to preserve some space.


## Synonyms

- [Synonyms Document](https://www.mongodb.com/docs/atlas/atlas-search/synonyms/)
- We need to have a collection defining synonyms.
- We can't do fuzzy search with synonyms.
- Two types of synonyms.
	- `equivalent`: The terms defined in array `synonyms` are all equivalent to each other.
		- E.g.: `["car","vehicle","automobile"]` are synonyms of each other and can be queried in place of each other.
		 ```json
		{
		  "mappingType": "equivalent",
		  "synonyms": ["car", "vehicle", "automobile"] 
		}  
		```
	- `explicit`: The terms defined in array `synonyms` are not equivalent to each other but equivalent to the term defined in `input`.
		- E.g.: `["maruti","tata","hundai"]` are type of `car`s.
		```json
		{
		  "mappingType": "explicit",
		  "input": ["beer"],
		  "synonyms": ["beer", "brew", "pint"]
		}
		```
- These kind of synonyms should be separate documents in single collection.
- Once we define this collection, we should (re)define our index to include synonyms as follows
```json
{
  ...
  "synonyms": [
    {
      "analyzer": "lucene.english",//OR WHATEVER TEXT ANALYZER YOU WANT TO USE
      "name": "{A_UNIQUE_NAME_FOR_YOUR_SYNONYMS}",
      "source": {
        "collection": "{COLLECTION_CONTAINING_SYNONYMS}"
      }
    }
  ]
  ...
}
```

## Performance and Sizing.

![[mongot(aka Atlas Search).drawio.svg]]