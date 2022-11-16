# Replication.
To read more..I have added [[Replication]] notes from M103.

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
- If a write concern is majority (or more than 1), primary syncs anything after T till a new write with majority write concern.
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

## Elections the simple version.

- Before calling an actual election, secondary calls for mock election where it determines whether it can win an election or not.
- Secondary misses the heartbeat from Primary
- Secondary can contact majority of other secondaries.
- That's where a secondary will propose an election and vote for itself.
	- With an election term (more like election version)
	- It will also state latest transaction time
- The eligible secondary needs to have a vote and needs to have a priority 0.
- Any node that can vote, will vote for first candidate whose message is received 
	- Only if the candidate is at same or higher transaction time
		- AKA Candidate knows much or more than receiving node.
	- Or Only if it have not voted for itself or other primary
	- If a receiver is primary, it will automatically step down at the time of actual election.

### Onboarding a new primary
- When a new secondary is selected, it goes through onboarding before it can accept writes.
- It checks if other secondary has higher transaction time?
	- If yes, it syncs the oplog from that secondary. [^1]
- It checks if any secondary has higher priority than itself?
	- It would call a "rigged" election so a higher priority node becomes primary and passes up up-to-date transaction

[^1]: It is only time data is copied from secondary to primary.

### Primary to secondary

A primary becomes secondary when
- It sees an election happening with greater version than that it was elected.
	- e.g. If it sees an election happening at `5` and it was selected in `4` it steps down
- Explicitly tell it to [[Replication#^stepDown|step down]].
- It can no longer see the majority of secondaries.


# Sharding

To read more. I have added [[Sharding]] notes from M103

## Common Sharding Challenges

- We may have badly distributed data
	- A shard which does more work than others is called "hot shard".
	- "hot shard" is not good, because if this shard is down, it impacts our app very badly.
- Adding new shard is itself a challenge
	- 