Mongod: Mongo daemon. Daemon is a program that is running, but we can not interact with it directly. Daemon usually has d appended to their name. Examples of daemons are like Redis server, docker daemon etc.

Mongod is the core server of database, handles connections, requests and persists data. To interact with daemon, we need specified client apps like CLI or Code libraries. Our commands are issued to client and client sends it to daemon to do actual process.

Each server part of replica set or shard will have their own mongod process running. In a multiple server deployment, we can configure our client to communicate with each of the mongod process as needed.

To start daemon, we need to run `mongod` command.

## Configuration

We can provide our configuration file or configuration options to mongod. If we run the command as it is, mongod will apply some default configurations.

### Default Configurations

- `Port`: 27017
- `dbpath`: /data/db (This is where the database, collections, documents and journals are stored)
- `bind_ip`: localhost (Only servers running locally in the machine are allowed to connect to db)
- `auth`: disabled

### Configuration Values

We can either provide the configuration values via CLI options or writing entries in YML file and provide it as a configuration. Here we have listed CLI options & yml entries together.
The format of most used configuration values and their description goes as below.

- `cliOptions` or `configurationFileSettingInYml`: Description of the option.

Here are the values being used.

- `dbpath` or `storage.dbPath`: The dbpath is the directory where all the data files for your database are stored. The dbpath also contains journaling logs to provide durability in case of a crash
  - If we want to point to new directory, mongod must have read and write permissions to that new directory since mongod will write data and journal to that path.
- `port` or `net.port`: The port option allows us to specify the port on which mongod will listen for client connections.
- `auth` or `security.authorization`: This flag enables authentication option (disabled by default). Regardless of values of this option, mongo shell in localhost can connect to mongod. Once the shell is connected, users and their access level can be configured from that shell instance.
  - To run mongod with auth option, run `mongod --auth` command.
- `bind_ip` or `net.bindIp`: This configuration allows exposure on said IP, the clients can only connect to said IPs. 
	- If I don't pass it, external clients will face problems to connect.
	- Multiple bind_ip s can be specified by separating them with comma.
- `fork` or `processManagement.fork`: Tells mongod to run as a real daemon instead of running it in terminal. To keep looking logs we need to `tail` the logs.
  - This option is not available in windows.
- `--config`: Tells mongod (or mongosh) to read configuration from a YAML file
- `-f`: Same is `--config`.

#### Location of config file

When mongod is installed via package manager, it creates a configuration file automatically. Here are the locations of the file per operating system

| Os | path |
|-----|------|
| macOS (Intel Processor)  | `/usr/local/etc/mongod.conf` |
| macOS (Apple M1 Processor) | `/opt/homebrew/etc/mongod.conf` |
| Linux (apt, yum, or zypper Package Manager) | `/etc/mongod.conf` |
| Windows (MSI Installer) | `<install directory>\bin\mongod.cfg` |

## Options and Mapping

- [Configuration options](https://www.MongoDB.com/docs/manual/reference/configuration-options/)
- [Command line options](https://www.MongoDB.com/docs/manual/reference/program/mongod/#options)
- [Mapping of Config File and Command line options](https://www.MongoDB.com/docs/manual/reference/configuration-file-settings-command-line-options-mapping/)

## File Structure

Like many databases, mongod stores the data and other access mechanism in various files. None of them are designed to be accessed by humans, their content only makes sense if they are read by mongod or mongo clients.

There are some `.lock` files (like `WiredTiger.lock` or `mongod.lock`) in the file structure. If these files are present and have some data in it, that indicates either a separate process working with MongoDB or there was an unclean shutdown. In both of the cases user may need to remove those lock files.

The `.wt` files, are related to data. The files start with `collection` are related to collection data and files start with `index` are data related to indices. When creating a fresh set of database, we may find some default collections and indices created already.

The `diagnostic.data` directory contains diagnostic information about mongo server. That diagnostic data does not contain any actual private data. These data is captured by FTDC (Full Time Data Capture) that includes information about server configuration, some statistics about server and the command line options used. To be able to use these files for any diagnostic, users have to provide them explicitly to MongoDB support engineers.

### Journaling

MongoDB write operations are by default buffered in memory and flushed at every 60 seconds, creating a checkpoint. Journaling systems also use write ahead journaling system. Journal entries are buffered in memory and then syncs the journal to disk every 100ms. Each journal file is limited to 100mb in size. WireTiger also flushes when all journal file has 2gb data. Flushing again creates another checkpoint.

When server unexpectedly crashes, there are some write transactions written in journal files. When mongod comes back online, WireTiger looks at existing data files and finds the identifier of last checkpoint and then searches journal file for the record that matches the checkpoint identifier and applies operations from journal files since the last checkpoint.

## Basic Commands

There are some helper commands.

### Shell helper groups

Methods available in MongoDB shell that wrap underlying database command.

- `db` shell helper: Interacting with database.
- `rs` shell helper: Managing `r`eplica `s`ets.
- `sh` shell helper: Managing `sh`arded cluster management and deployment.

## Some useful shell helpers

- User Management.
  - `db.createUser()`
  - `db.dropUser()`
- Collection Management
  - `db.renameCollection()`
  - `db.{collectionName}.createIndex()`
  - `db.{collectionName}.drop()`
- Database Management
  - `db.dropDatabase()`
  - `db.createCollection()`
- Database Monitoring
  - `db.serverStatus()`

## Logging

Two types of Logging. Process Log and ...

### Process Log

Process logs collects activity in one of the following components.

- `ACCESS`(`accessControl`): Messages related to Access control.
- `COMMAND`(`command`): Messages related to DB Commands
- `CONTROL`(`control`): Messages related to Control activities like initialization
- `FTDC`(`ftdc`): Messages related to Diagnostic Data Collection mechanism.
- `GEO`(`geo`): Messages related to parsing of geospatial shapes (Lat long and places)
- `INDEX`(`index`): Messages related to indexing operations
- `NETWORK`(`network`): Messages related to networking activities such as accepting connections
- `QUERY`(`query`): Messages related to queries including query planner activities
- `REPL`(`replication` namespace): Messages related to replica sets such as initial sync & heartbeats
- `SHARDING`(`sharding` namespace): Messages related to sharding.
- `WRITE`(`write`): Messages related to write operations

These messages have verbosity value attached to it. `-1` means inherit from parent. Default value `0` means informal level. Verbosity can go from 0-5. Higher the verbosity level, more messages will be printed related to that component.

## Profiler

Profiler is used to measure db performances. Most of the time logs don't reveal whole story. Profilers are enabled at database level so profiling can be done separately.

When enabled, profiler writes data for all operations of given db and creates a new collection called `system.profile`. This operation will do profiling and hold data on CRUD operations, Admin Operations and Config Options. This profiler is a capped collection, the capping is 10MB max. ^systemProfileCollection

Three settings values.

| Value | Meaning |
| ----- | ------- |
| `0` (Default) | Profiler is off and does not collect any data |
| `1` | Profiler collects data for operation that takes longer than value of `slowms` |
| `2` | Profiler collects data for all operations |

- We can get current profiling level with `db.getProfilingLevel()`. Which returns following object `{ was: 0, slowms: 100, sampleRate: 1, ok: 1 }`.
  - Here `was` is previous (or current) profiling level
  - `slowms` indicates the maximum time query should take before it is considered slow. ^slowMs
- We can set profiler level with `db.setProfilingLevel(profileLevel, {ProfileParams})`.
  - We can set `slowms` using this method call.
- Profiler adds additional overhead for all operations, so it's not recommended to keep this turning.
- Recommend to read logs if using 4.4+ #recommendation 

## Authentication & Authorization

Security concepts:

Used for Authenticating user.

- `Challenge`: Something that proves the authentication of user, AKA `who is the user?`, `how does the user prove his identity?`.
- `Response`: The response from user regarding the challenge, AKA `The username and password` or `federated login`.
- `Validation`: The logic from server who validates the response for the challenge, if the response is validated, the user is authenticated, if not server keeps providing different challenges.

Authorization is what are the privileges of user?.

**Authentication answers and validates who the user is** and **Authorization answers what kind of access this user has**. For example, by providing my GitHub credentials, GitHub know who I am, which is called authentication. And they have stored all data related to which repository I can access, which is called Authorization. These authorization data contains all open source repositories, repositories I have created, and private repositories I am allowed to see.

### MongoDB authentication mechanisms

#### Available in all versions

- `SCRAM`(`S`alted `C`hallenge `R`esponse `A`uthentication `M`echanism): Default authentication mechanism.
  - Here MongoDB provides some challenge that user must respond to.
  - Equivalent to Password Authentication
  - Client requests a unique value (Nonce) from server
  - Server sends nonce back
  - Client hashes password, adds nonce and hashes hashed password+nonce again
  - Server has hashed password saved
  - Upon receiving end, server picks hashed password and hashes it + nonce it has sent to client.
  - If both the hashes are equal, the login is successful.

- `X.509`: Uses X.509 certificate for authentication.

#### Available in enterprise versions

- `LDAP`(`L`ightweight `D`irectory `A`ccess `P`rotocall): Basis of Microsoft AD.
	- LDAP is sending plaintext data by default, TLS is required to make it secure.
- `KERBEROS`: Powerful authentication designed by MIT.
	- Like `x509` certificates but they are very short lived.


#### Available only in Atlas
 - `MONGODB-AWS`: Authenticate using AWS IAM roles.

### Intra-cluster Authentication

- Two nodes in a cluster authenticate themselves to a [[Replication|Replica Set]]. They can either use `SCRAM-SHA-1` by generating Key file and **sharing key file between the nodes**, (the approach used in replication labs).
	- If they use this key file mechanism user will be `__system@local` and password will be generated keyfile
- Or they can authenticate using `X.509` certificates which are issued by same authority and **can be separated from each other**.

## Authorization

- RBAC is implemented
  - Each user has one or more roles
  - Each role has one or more privileges
  - Each privilege represents a group of actions and the resources where those actions are applied
  - E.g. There are three roles, `Admin`, `Developer` and `Performance Inspector`
  - `Admin` has all the privileges.
  - `Developer` can create, update collection, indices and data. They can also delete indices and data. But can not change any performance tuning parameter.
  - `Performance Inspector` can view all the data and indices, can see and change performance tuning parameter.

### Localhost Exception

`Localhost Exception`, even with auth enabled MongoDB doesn't provide any default users, we have to create one ourselves. So `Localhost Exception` is created so that we can connect to `mongosh` running in the same machine of `mongod`, and create a user.

Localhost Exception doesn't apply in following scenarios.

- Once a user is created `Localhost Exception` doesn't apply, so it's desirable to create first user as admin privileges.
- If we define any authentication mechanism, `Localhost Exception` doesn't apply when we connect to `mongod` or [[Sharding#`mongos`|mongos]] using their replica set or shard URLs, localhost exception only applies if we connect to that specific node.

>[!warning]
>With Localhost exception we can create first user or role, as soon as we create either user or role, our localhost exception is disabled.



### Role structure

- Specific DB and specific collection
  - `{db:'databaseName',collection:'collectionName'}`
- All Databases and all collections
  - `{db:'',collection:''}`
- All Collections in given DB
  - `{db:'databaseName',collection:''}`
- Collection name in any Database
  - `{db:'',collection:'collectionName'}`
    - This means this privilege gives access to given collection name in all databases.
- Cluster Resources
  - `{cluster: true}`

If someone needs to be given access to delete and update accounts collection in any database the access this privilege looks like.

`{resources: {db:'',collection:'collectionName'}, actions:["delete","update"]}`

Role can inherit from other roles.

We can allow defining network restrictions (Source IP or Destination Address) in Role definitions.

### Built in Roles

MongoDB provides built-in roles. Organized in 5 groups.

- Database User
  - `read`
  - `readAnyDatabase`*
  - `readWrite`
  - `readWriteAnyDatabase`*
- Database Administration
  - `dbAdmin`
  - `dbAdminAnyDatabase`*
  - `userAdmin`
  - `userAdminAnyDatabase`*
  - `dbOwner`
- Cluster Administration
  - `clusterAdmin`
  - `clusterManager`
  - `clusterMonitor`
  - `hostManager`.
- Backup/Restore
  - `backup`
  - `restore`
- Super Admin
  - `root`*

The roles are assigned to user on database level. A user can have different roles on different database without affecting access rights with each other.

The roles marked with `*` are applicable to all databases.

We can also create some custom roles to fit our needs.

### User Defined Roles

User defined roles can be created on a given database, and can only inherit roles created in same database. If we need to share user defined roles, **global role**, between the databases, such roles must be created in `admin` database.

- To Create a role, call `db.createRole({role,privileges:[{resource:{db,collection},action}],roles:[{role,db}]})` command in the database where you want to create the role.
  - Here `role` stands for role name
  - `privileges` denote which privileges should be there in current role.
  - `roles` define which roles we are inheriting from.

Here are the list of available [resources](https://www.mongodb.com/docs/manual/reference/resource-document/) and [actions](https://www.mongodb.com/docs/manual/reference/privilege-actions/) to create or update user defined roles.

- To add new roles to users call `db.grantRolesToUser` method. Calling `db.updateUser` with new roles will override all existing roles.
- To update the role, there are three ways.
  - Call get role, update the `privilege` document and call `db.updateRole` command. This is more powerful because it will replace `privilege` entirely.
  - Call `db.grantPrivilegesToRole` command, we can add new privileges (new actions and/or new resources).
    - Similarly, `db.revokePrivilegesFromRole` command revokes existing privileges from existing role.
  - Calling `db.grantRoleToRole` allows us to inherit new role(s) for given user defined roles.
    - Similarly, `db.revokeRolesFromRole` allows us to remove existing role inheritance.

To see what each role can do on given database. Run following command.

```sh 
db.runCommand( { rolesInfo: { role: "ROLE_NAME", db: "DATABASE_NAME" }, showPrivileges: true} )
```

This command will return following interesting properties.

- `db`: In which database the role is being used
- `role`: Name of all roles
- `roles`: (?)
- `privileges`: Lists out all privileges that this role defines.
- `inheritedRoles`: All roles which are inherited by given role.
- `inheritedPrivileges`: privileges coming from inherited roles.
- `isBuiltin`: Boolean flag denotes if this role is built in or not.

DB Owner role is a super admin for given database, that role has privileges of `readWrite`, `dbAdmin` and `userAdmin` roles for that database.

## Server tools

We have `mongod`, which is daemon and `mongosh` (or in older version `mongo`) which is CLI Application that connects with mongo daemon. There are other tools to help us.

To get all available CLI tools in mac, run following command.

```sh
find $(dirname $(which mongod)) -name "mongo*"
```

This will first run `which mongod` command, this will give directory location from where `mongod` is running, and then parse this as `dirname` which we can run in find command to get all CLI starting with mongo.

Here are some tools and their description.

- `mongostat`: Quick statics about `mongod` or `mongos` process.
- `mongorestore` and `mongodump`: Import and export dump files from MongoDB collection. These files are in `bson` format. These exports also have metadata file that indicates collection and indexes.
- `mongoexport` and `mongoimport`: Import and export data from MongoDB collection. These files are in `json` or `csv` format (This will be decided by `--type` parameter). Defaults to `stdout` to export or `stdin` for import. To use files use `--out` parameter with `mongoexport` or `--file` parameter with `mongoimport`. These export doesn't have metadata files. So MongoDB has to be told where to import the collection (defaults to `test.{fileName}` collection)

>[!tip]- BSON and JSON export/import
>For `bson`, use `dump` and `restore` (DB Specific terms), and for `json` use `export` and `import`.(Data specific terms)
>

