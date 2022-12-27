![[Optimizing Storage#^onlineArchiveExplain]]

# Multi Region Multi Cloud Analytics Direct Read.

- [Docs](https://www.mongodb.com/docs/atlas/reference/replica-set-tags/)
- Use `readPreferenceTags` in connection strings.

# Cloud Provider instance size Options.

- [Internal Doc](https://wiki.corp.mongodb.com/display/cs/Cloud+Provider+Instance+Size+Specifications)

# Data Transfer Costs

- In ideal cases DTC should not go beyond 10% of their total costs
- Charged for Data transfer between Atlas To other nodes
- Order of cost (From Lowest to highest) is
> [!TIP]+ Cloud Cost Order
> ## AWS
> - Same Region
> - Other Region
> - Outside of any AWS Cloud
> ## GCP
> - Same Region but different zones
> - Different Regions but within USA
> - Different Continent
> - Same Continent (Excluding USA) but a different region
> - Outside Google Cloud
> ## Azure
> - Between Availability Zones
> - Using in-region VNet Peering
> - Different region (Depends on Geographic location of source code?)
> - Cross region VNet Peering.

- Tips for reducing the cost:
	- Queries should not
		- Re-read the data which is already available on the client
		- Re-write existing data 
	- Whenever possible, ensure that queries originate from same cloud provider and region 
	- For cross region queries
		- Read Queries have [nearest read preference](https://www.mongodb.com/docs/manual/core/read-preference/#mongodb-readmode-nearest)
		- Use write queries from highest priority region
	- Use aggregation framework to pre-process the data and use project to limit the fields you need.
	- Ensure that your driver uses wire protocol compression.

# Archive

## How Atlas Archives data
- **Job**: A query to determine the document that matches archiving criteria
- Atlas runs it every 5 minutes. It checks if the output is 2GB or greater.
- If not it expands the job interval by 10 mints. (Next job will run after 15 mins, 25 mins, 35 mins, 45 etc...upto 12 hours AKA 720 mins)
- If the job interval reaches to maximum, or size or document numbers reaches threshold, above limit is reset to 5 minutes.
- Index Is needed unless the QTR is < 10.
- 

# Private Endpoints

1. 