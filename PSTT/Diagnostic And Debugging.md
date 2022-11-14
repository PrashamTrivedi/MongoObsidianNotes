Here are some notes and bullet-points about diagnostic and debugging. Note that this document is not comprehensive and will be updated on time to time.

## `db.serverStatus`

Used to get the diagnostic information about server.

- `db.serverStatus().opcounters`: Number of operations after the server has started. [Read more](https://www.mongodb.com/docs/v5.0/reference/command/serverStatus/#std-label-server-status-output).
- `db.serverStatus().connections`: Number of active and allowed connections with server. [Read More](https://www.mongodb.com/docs/v5.0/reference/command/serverStatus/#connections)
- `db.currentOp()`: List Current Operations running in server. [Read More](https://www.mongodb.com/docs/v4.4/reference/operator/aggregation/currentOp/)
    - Users with `clusterMonitor` can use `currentOp`.

## Misc Points

- Without mentioning [[Mongod#Authorization|`--auth` option in `mongod`]] we can not enforce the roles.
- You can check which flags are used to start mongod using following command
```js
	db.adminCommand({getCmdLineOpts: 1})
```


## Tips From Gobind
- The Simpler the query the better the schema is #gotchas #recommendation 
- Slow Queries: Query, Schema & Indexes (Check all 3 things together)
- We can't simplify everything, prioritise them.
- Pick those queries first which are business critical and/or most used.
- Use 70/30 rule, mostly majority workload (~70%) will be served by (~30%) of queries, prioritise them.
- Avoid operators
	- Negation Operation `$ne` `$not` `$nin`
	- Regex Operator
		- ![[Optimizing Storage#^regexIndex]]
- If your keyExamined number is greater, this is effectively a `COLLSCAN` in memory or unchecked range.
- `$or` should use explicit query for every sub-clause.
- `$elemMatch` bug: It will only pick index for first attributes.
- `$in` is equality if array size <=200, otherwise it's taken as range. #documentTodo 
- Workload isolation: Isolate other heavy queries to isolate to secondaries so primary won't be contended.


TODO: exists and or check with explain