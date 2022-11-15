
## Slow Query

^fa88a1

What is slow query?:
- A query that takes more than desired time to complete and viable.
- Blocking critical (or useful) user operations.
- A Bad UX.
-------------

## Slow Query in MongoDB

- A query that takes more than desired `slowms` parameter are considered Slow queries.
- By default `slowms` is `100ms`. 
- Can be changed using `db.setProfilingLevel(profileLevel, {slowms:NEW_SLOWMS_VALUE})`

--------------

## Identifying Slow Query - 1

- Get Database logs.
	- Using `db.adminCommand({getLogs:"global"})`
- Reading log files.
	- Require access to configuration file or need to run `db.adminCommand({getCmdLineOpts: 1})`  
- Logs are arranged in JSON after 4.4. 


---------

## Identifying Slow Query - 2

- Profiler and their default values 0,1 and 2
- We can turn on profiler using `db.setProfilingLevel(1)`
- This will log all slow queries to `system.profile` collection.
- Use profilers wisely.

--------

## Slow Queries: Why?

- General rule: RAM is faster than Disk.
- Reading from disk makes operation slower.
	- Missing Index
	- Wrongly utilised or under utilised index.
	- Document too large to fit in RAM

------

## Index

- Most of the times we don't need whole collection. We just need limited number of records.
- To get just one/some user(s) by their email, scanning entire collection of users is not optimised. 
- Thus an Index should be created which maps the key and values to document where the data resides.

---------

## Query Explanation - 1 

- use `.explain()` with desired operation to get the data
- Check `winningPlan`.
	- `COLLSCAN`: Scanning entire collection
	- `IXSCAN`: Using index 
	- `IDHACK`: `_id` was used.

-----

## Query Explanation - 2

- Use `.explain("executionStats")` to get more statistics.
- Check `executionStats` object.
	- `nReturned`, `totalDocsExamined`, `totalKeysExamined`: Total index keys examined
- Ratio of `nReturned` to `totalDocsExamined` should be closer to 1.

---------

## Query Explanation - 3

- Sometimes your desired Index are not used in planner.
- Ways to check
- Use `.explain("allPlansExecution")` shows failing plans and statistics.
- Use `.aggregate([{$indexStats: {}}])` determines index uses.
- Use `.hint(index).explain("executionStats")` with desired index and compare it winning plan.

-----

## Tips to avoid slow query.

- Use index wisely. 
	- Use Index Prefixing. (AKA most searched fields should come earlier in index)
- Follow ESR rule. 
- Model your collections properly.(Different topic altogether)
- Always examine `explain` output and tweak accordingly

------

# _May the force be with you_
