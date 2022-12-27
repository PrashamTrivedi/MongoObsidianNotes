![[Optimizing Storage#^onlineArchiveExplain]]

# Multi Region Multi Cloud Analytics Direct Read.

- [Docs](https://www.mongodb.com/docs/atlas/reference/replica-set-tags/)
- Use `readPreferenceTags` in connection strings.

# Cloud Provider instance size Options.

- [Internal Doc](https://wiki.corp.mongodb.com/display/cs/Cloud+Provider+Instance+Size+Specifications)

# Data Transfer Costs

- In ideal cases DTC should not go beyond 10% of their total costs

## AWS
- Charged for Data transfer between Atlas To other nodes
- Order of cost (From Lowest to highest) is
	- Same Region
	- Other Region
	- Outside of any AWS Region
- 

# Private Endpoints

1. 