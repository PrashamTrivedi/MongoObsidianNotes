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
	- Adding a new shard is simpler, but they are added as empty state
	- Balancer will run the data and move the chunks, but it can take days of weeks.

### Managed Sharding

- It's a technique to avoid sharding pitfalls.
	- Requires planning and efforts
	- Requires code changes
- Managed sharding works better when we have compound index.
	- Calculate hash of one of the field with one of the passively compound number and have a modulo on that number.
		- That passively compound number should be `2*3*4*5` or `2*3*4*5*7` 
		- This gives an evenly distributed number with good enough cardinality and no monotonically increasing value
	- Then the shardkey is hash of original field and other field instead of two original fields.
	- At the time of the query, this hash value has to be queried and included in query.
	- It's also advisable to add index on original fields as well for optimisation.

### Presplitting the data

- Balancer will split the data itself
- MongoDB allows us to explicitly split chunks
- We should turn off the balancer.
- We will define shard limits and MongoDB will split the chunks as we insert the data
- Shards will split instantaneously in newly added shards if shard keys are planned correctly.
	- e.g. If shard key is timestamp, and on of our chunks define min timestamp is week from now, any data which contains this future timestamp will be written in new shard.
	- Old shards will be effectively only do updates and deletion from that point forward
- With managed sharding we can split 2,3,4,5 or 7 shards on the fly.

### Pros and Cons

- Pros
	- Scale out instantly
	- Always well balanced
	- Can downgrade older shards
- Cons
	- Requires some sort of planning
	- We need to scale out atleast twice
		- If I have 3 servers, my load can't be served with 4 servers instantaneously. 6 will do justice
	- Needs additional coding from developer
	- Shard keys must be in query to take advantage

## Zone based sharding
- Storing some shards in different zones (aka locations)
- Telling the balancer to store something in specific shards
- Used to store specific country's data in specific shard
	- Useful for GDPR compliance
	- Better for latency

### Sharding for parallelism

- Unusual but powerful case
- Useful for reading data
	- But not for OLTP or normal queries
- Sometimes many shards in one server (called microsharding), works where
	- All data can fit in RAM
	- Small number of power users, means more CPU is available but less users
	- We know how to write aggregation for parallelism.

# Security

## Keys and PKI

Originally from [here](https://notes.prashamhtrivedi.in/saa/encryption.html)


### Symmetric Encryption
- Same key is used to encrypt and decrypt the data.
- More easy to compute.
- Key must be shared between parties. That makes the encryption useless.
- Good for local encryption or disk encryption.

### Asymmetric Encryption
- Different key is used to encrypt and decrypt the data.
- Two sets of keys: public key and private key.
- Public keys are shared openly in the world.
- Private keys are stored with the receiver.
- Public keys are used to encrypt the data, and private keys are used to decrypt the data.
- Signing
	- When sending message, sender signs that message with the private key, and receiver can verify the sender using sender's public key.
	- This is inverse of what discussed above, in signing encryption happens with private key and decryption happens with public key.
		- Like when A is sending message to B.
		- A encrypts the message using B's public key and signs this with A's private key.
		- When B receives this, only B can read this message with his private key, and verify that it's sent by A by decrypting the signature using A's public key.

### Certificate
- A certificate says
	- Who issued it
	- When it was issued
	- To whom it is issued
	- The public key of the issued
	- How long it will be valid
- To request the certificate
	- We create a certificate that mentions who we are, what we want to do and our public key
	- We sign it using our private key
	- We send it to Certificate Authority, who digests it, encrypts it and signs it.
## Public Key Infrastructure

- There are top level CAs whose information resides in OS and Browsers
- They may issue certificate to be a CA for specific subdomain
- There is a chaining involved
	- E.g. My website can be signed by CA XYZ, their Certificate (which has CA rights) is signed by CA JKL who has a Certificate with CA rights which is signed by Verisign. 

![[Mongod#MongoDB authentication mechanisms]]

### What to use and when

- For humans, individual logins are preferable
	- LDAB/Kerberos is preferable because there are automatic revoking based on rules
	- SCRAM is secure but not centralised, we need to add same users manually everywhere
- For servers/services
	- Should be hidden by humans
	- Kerberos is great for windows service
	- On Atlas, Atlas with AWS IAM is preferrable
	- Generate a password at install time
	- Or expose a password using a secret store.

## Authorization

![[Mongod#Authorization]]

### Document level security
- Roles give us collection level security