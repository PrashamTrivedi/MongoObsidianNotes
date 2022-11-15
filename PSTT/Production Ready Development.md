# Replication.

## Secondary Servers

- ![[Replication#Other secondary node types]]
	- PSTT Special: Delayed and Hidden members considered legacy (In Mongo University Course they are covering old courses) and should not be used
		- Hidden members were there to isolate read workload from regular app and for other (analytics type) workloads.
			- Users can specify `readPreference` as `secondaryPreferred` to divert the workload to a secondary.
		- Delayed members are there to provide a backup, but 
			1. It has to have sufficient window to be available for proper [[Operational Skills#Basic Backup Options|Backups]]
			2. If your delay is lesser, it won't work effectively as a backup, if it's too higher, it will miss oplog sync and take more time for RTP
		- Use other modes of backup available. A short summary is below
			- ![[Operational Skills#Backup methods]]

## Drivers and Replica Sets
- Drivers can keep track of the [[Replication#Replica sets topology|topology]].
- It knows where and how to route request. 
	- ![[Replication#^rsIsMaster]]
- Reads can go to Secondary using [[Replication#Read Preferences|Read Preferences]]
>[!caution]
>When upgrading the server, it's important to check driver compatibility.

## Oplog

- Apps are written to primary
- Primary applies changes at time T and records it's changes to Operation log
- Secondaries are observing this Oplog and read changes upto time T 
- Secondaries apply new changes upto T themselves
- Secondaries also record their own Oplog like primaries
- Secondaries request time after T
- Primaries know latest seen T for each secondary 
- Oplog is Capped and read only ([[Replication#^oplogCapped|Read more]])
- Oplog commands are idempotent, instead of running command as it is, oplog are storing command as state of data.
	- e.g. Instead of storing `{$inc:{a:2}}` Oplog Stores `{$set:{a:4}}` provided in previous statement a is 1.
	- Or instead of `{$push:{x:1}}`, `{$set:{"x.2":1}}`.
	- Connected notes about idempotency are [[Replication#^oplogIdempotent|here]] and [[Design Skills and Advanced Features#Idempotent Operations|here]]

>[!note]- Oplog Internals
>Oplog internals are noted [[Replication#oplog.rs|here]]

### Initial Sync

- ![[Replication#^oplogRecovery]]
- Recent Changes #version #gotchas 
	- In 4.2/4.4
		- Can auto restart from non transient error
		- Resumable on transient error
		- Can copy oplog while initial sync
		- Can use secondary as source.

