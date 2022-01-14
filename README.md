# Neo4j deloyment investigation
## Neo4j cluster architecture
![image](https://user-images.githubusercontent.com/1872337/149515280-5d269bae-ef50-44e3-a1d8-fa88df0b2769.png)

Primary servers are used for writes, secondary servers - for reads.  
Only primary servers adds to fault tolerancy. 
Number of primary/secondary servers can be scaled independantly, depending on the kind of workloads(read/trasactional).  
The common setup is 1 primary server (w/o fault tolerancy and possible data lost), or 3/5 primary(core) servers. Amount of read replicas depends on the 'read' load.  
  
Neo4j uses binary log replication to sync data, causal consistency for client apps and discovery (in caes of k8s deployment, it will use k8s services to gather node\s IPs)  
In case of multiple core instances, one will be a leader and others are followers for specific database. This means that a Core instance may act both as Leader for some databases, and as Follower for other databases.  


## AKS deployment
![image](https://user-images.githubusercontent.com/1872337/149520470-050d5923-c93e-4cf4-b5a8-b23682bd1dc5.png)



Each node should have own persistent storage (Azure disk). Suggested storage type is _Premium_LRS_ at least. We create AzureDisks and then use it from k8s neo4j nodes through k8s PersistentVolumeClaims.    
The easiest way is to use helm v3 charts. 
There are helm charts to deploy core nodes(__chart neo4j/neo4j-cluster-core__), as well as needed amount of read replicas(__neo4j/neo4j-cluster-read-replica__). 

![image](https://user-images.githubusercontent.com/1872337/149521107-6b309911-6055-4d82-a462-6242ac0bb80b.png)

Additionally, we might need to deploy __neo4j-cluster-headless-service__ to have an access from inside of k8s, and __neo4j/neo4j-cluster-loadbalancer__ to have an access from the outside of k8s cluster.  

k8s resources used: ConfigMaps, StatefulSet for core nodes, Deployment for replicas, Service, ServiceAccount linked to cloud IAM etc

## Autoscaling
Scaling read replicas is relatively easy using HorizontalPodAutoscaler based on CPU load (needs minimum configuration)  
Scaling core members __is not recommended__ (as it involves rebalancing the cluster and replicating the data):
> Automated scaling applies only to read replicas. At this time we do not recommend automatic scaling of core members of the cluster at all, and core member scaling should be limited to special operations such as rolling upgrades, documented separately. 

## Monitoring
Neo4j supports prometheus metrics(regular k8s flow) - no problems expected  

## Multi tennancy

### Security

### side notes
Clustering available only for Neo4j Enterprise Edition (paid)


### resources
Official documentation is pretty good
https://neo4j.com/docs/operations-manual/current/clustering/introduction/
https://neo4j.com/labs/neo4j-helm/1.0.0/operations/
https://neo4j.com/docs/operations-manual/current/kubernetes/quickstart-cluster/server-setup/
