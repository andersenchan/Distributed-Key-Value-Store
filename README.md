# Distributed-Key-Value-Store

Message me to see the code.

## Architecture

### Database

#### Partition
Each node is responsible for a certain subset of the shorts. For example, node 1 is responsible for shorts beginning with a to g. This mapping is determined based on the number of nodes.

#### Replication
The first node has 2 databases: database A (database.txt) and database B' (database_backup.txt). Database A is database partition for storing the data subset that the particular API server is responsible for. Database B' is for storing a backup of the second node's database B (In particular, the node is just the next node in our list of nodes). This is in case node 2 with database B goes down, node 1 can take over node 2's responsibility and catch it back up when node 2 is back up again.

Node:				1	2	3	4
Primary Database:	A	B	C	D
Backup Databasse:	B'	C'	D'	A'

Another implementation is to store the database backup directly in the proxy server. However our nodes are supposed to be the ones doing the work and proxy server just redirects. And in the real world, the proxy server can go down just like any other node so there is nothing special about storing the data there.

#### Database Snapshot
As mentioned in Replication, we store a backup of the database in another API server. If the Sentinel detects that the node is down, it will try to restart the node and then give it the database B' to overwrite database B (since we assume database B is outdated). Also it will give database C to it so its backup database C' can be updated.

### API Server

API Server is multithreaded

We saw no point in making each node have its own tasks (ex one node for reading, one for writing, etc.), since it makes the system less flexible, and we gain no performance benefits (at least from our perspective).

### Proxy Server:

#### Monitoring (Sentinel):
We created a thread within the proxy server that continually pings nodes and sets their statuses accordingly. These statuses are printed to console. The proxy server will change its behaviour on which node to send the request to based on the node's status. For example, if the node is down, then it will use the backup node to serve the request.

We also wrote a script to check proxy server down and bring it back up if its down (./scripts/checkProxyServer.bash), but it's not a part of the assignment as we assume the proxy server will not go down.

#### PUT Redundancy
Proxy server writes to 2 nodes. The first write is for the API Server handling that particular data subset. The second write is for the database backup, stored on an alternate API server.

#### GET Redundancy
Proxy server reads from 1 node. Usually the API server that is supposed to handle that data. If that one is down, then it reads from backup database stored on the alternate API server.


## Running The System

#### Database
When we start the service, we assume the database is partitioned correctly as described in the architecture. If this is a problem, we have a script to delete all the databases in each node. Empty databases satisfy the requirements for our architecture.

#### Changing Hosts
Edit the hosts.txt file in the repo and add all the nodes's hostnames that you want to include.
Our system will automatically read from this file to find out which hosts to use during deployment.

Note: our scripts automatically compile the code and kill the service if it's running.

#### Running Service
In the machine that you want to run the proxy server in, the script ./scripts/startService.bash starts the service by starting the proxy server and SSHs into each node and runs ./scripts/startNode.bash. 

#### Running Individual Components
./scripts/startProxyServer starts only the proxy server and no nodes
./scripts/startNode starts only one node

## Testing The System

in ./performanceTesting, there is ./readTest.py and ./writeTest.py and ./loadTest

In ./scripts, there is a testNodeHealing.bash script which 
1. takes one node down and tests to see if requests can still be made
2. Starts the node back up, and checks if the node is revived
3. Checks the revived node's database for consistency

### Results

#### Optimal concurrent requests
We have pictures of graphs (x = number of concurrent requests, y = speed) for GET and PUT requests, and the optimal number of concurrent requests seems to be 2 to 3.

#### Comparison with initial system

We also have ab output for initial system (starter code), a multithreaded version of the intial system, and our final system.
Initial - 41ms/request
Initial (Multithreaded) - 1ms/request
Final - 53ms/request

If we are focusing only on improving latency, then we can implement the multithreaded version of the service and we would recieve a huge improvement.
However, our final version sacrifices speed to increase fault tolerance and data scalability.


## Monitoring/Debugging

#### Sentinel

Sentinel prints host statuses to the console in an interval.

#### Scripts

Most of our scripts write output to a log file with the same name as the script.

#### API Server

API Server logs output to a text file, since it is run in the background and cannot print to console. Printing to a text file is better anyways in terms of persisting the server log.

## Analysis: 

### Scalability:

#### Increased data

Since we partitioned the data, it is easier to add more data and not hit the size limit of the machine (since we can just add another node to store the data in instead of storing it in an existing full machine). This would not be possible if we did not partition the data.

#### Increased number of requests/ Improving speed

##### Multithreaded

Each node can handle multiple requests since each API server is multithreaded.

##### Adding nodes

We can add more nodes, and since the data is partitioned, the proxy server only has to talk to one (or two) nodes at a time. If we did not partition, the proxy server would have to write to all nodes, for example, which would not be scalable.

### Latency/Thoroughput:

In ./performanceTesting, there are files detailing the results of our benchmarking.

Our implementation is less fair in the case that a certain short is more likely than the rest. In that case most of the load will be experienced by the node that has the data partition for that request parameter. Our original round robin implementation of choosing the hosts would be more fair. An ever fairer approach is to find out how long a GET and PUT request takes and then balance the requests between the nodes based on this information.


### Availability:

System remains available and consistent so long as:
1) the load balancer is available and functioning
2) no 2 sequential nodes are down

Our system is always available since it returns an HTML page displaying the result of the request. Even in the case that both the API Server and the backup API Server are down, we send an HTML page saying we could not perform the GET/PUT.

Of course, we assume that the load balancer / proxy server is always running. Since our system is not partition tolerant.
			
### Consistency:

System remains available and consistent so long as:
1) the load balancer is available and functioning
2) no 2 sequential nodes are down

If 1 node goes down, or 2 non-sequential nodes, we still have the backup database to read and write from. Then, when we detect a node is down, we have a script to catch its database up with the backup database.

If 2 sequential nodes go down, we cannot write/read from database or backup database, which means no GET/PUT requests can be completed correctly (meaning our system does not function correctly so it is not partition tolerant).

It still may be possibly to have inconsistency outside of the above case (2 sequential nodes going down), because when you PUT, you write to 2 nodes, the original node and backup node. One of them can fail. Which means only one gets the write, causing inconsistency. We currently did not implement a way to combat this. For example, we could have a thread monitoring diffs in databases and launching a script to write whatever key/value pairs are missing, which means eventually we would become consistent. We can ignore this case though since we know our system is not partition tolerant, so if we dont lose any packets our system will be consistent.


### Durability:

We do not store the data in RAM, protecting against any potential data loss when a node is taken down.

#### Data Partitioning 
Since we partition the data, when a node goes down, (n-1)/n of the data is still available

#### Data Replication
Furthermore, since we add a backup of the database in a separate API Server, we still have access to the data so 100% of the data is still available.
Of course, we only have 1 backup so if both the original node and backup node is down, then we are back to only having (n-1)/n of the data available (since we do not have 2 nodes store backups of each other unless there are only 2 nodes, in which case none of the data will be available).
We can withstand n/2 node fails with no data loss and full capability, assuming no two consecutive nodes are taken down.

### Fault Tolerance:

#### Healing

If one of our nodes go down, we will be able to detect it because of the sentinel. In this case, a script will run to attempt to restart it.

Sentinel pings constantly so we know quickly when a node is down. Hopefully it will bring it back up before a client makes a request to it, so they don't notice.

To combat proxy server going down, We did write a script to keep checking proxy server status and bring it back up if it's down (called ./checkProxyServer.bash).

#### Partition Tolerance:

Our system is not partition tolerant.

if 2 sequential nodes go down (or. we cannot communicate with them), then we will not be able to read/write with their data partition.

Furthermore, if our proxy server goes down, then we lose the entire service.

We do have scripts to restart but during the time that proxy server or nodes are restarting our system will not be functional.






