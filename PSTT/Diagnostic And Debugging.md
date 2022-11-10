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