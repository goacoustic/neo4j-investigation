# Neo4j deloyment investigation
## Neo4j cluster architecture
![image](https://user-images.githubusercontent.com/1872337/149515280-5d269bae-ef50-44e3-a1d8-fa88df0b2769.png)

Primary servers are used for writes, secondary servers - for reads.  
Only primary servers adds to fault tolerancy. 
Number of primary/secondary servers can be scaled independantly, depending on the kind of workloads(read/trasactional).  
The common setup is 1 primary server (w/o fault tolerancy and possible data lost), or 3/5 primary servers. Amount of read replicas depends on the reads load.

