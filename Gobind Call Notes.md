# Consult Process

## Q: Performance issue on atlas.
### Thought Process
- A Good CE would also share the knowledge which solves the problem.
- Make them calm.
- Always give options, all the options should be beneficial to us.
	- 1. Do I have a problem?
	- 2. Where Do I have a problem?
	- 3. How to fix it?
- Seeing couple of matrix is never enough, telling a story using these matrix is always effective.
>[!tip] Start with hardware, correlate with MongoDB graph
- DISK IOPS: Rate of throughput
	- How much your tap can output the water per second.
	- How much your factory can output.
- DISK UTILISATION: How much my disk is used to achieve this IOPS.
	- How many buckets you need to fill these water
	- How many workers you need to achieve this output
- Tickets available should not fluctuate...


## Tips of reading performance metrics.
- First Get the configuration of current cluster
- Use DISK IOPS to measure transfer rate. 
- Beware of IOWAIT, they are indicators that your disks are busier then they should be
- Also take Max normalised CPU in account. 
	- Normalised CPU tells you average usage
	- Max Normalised CPU tells you the maximum usage (Will tell you if a scale up should happen or not)
	- If there is more CPU utilised then currently available there was a scale up.
	- You can confirm it from Activity Feed.
- Check if majority of load is happening on one node or not. If yes, all reads are also being done from primary. 
- All the find operation for updates are done on primary regardless.
- 
