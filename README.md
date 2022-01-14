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
> A store copy is a major operation, which may disrupt the availability of instances in the cluster.  
> Automated scaling applies only to read replicas. At this time we do not recommend automatic scaling of core members of the cluster at all, and core member scaling should be limited to special operations such as rolling upgrades, documented separately. 

## Monitoring
Neo4j supports prometheus metrics(regular k8s flow) - no problems expected  

## Multi tennancy options
1. Starting from version 4.0, Neo4j supports Multi Tenancy.
> we can create and use more than one active database at the same time. This works in standalone and causal cluster scenarios and allows us to maintain multiple, separate graphs in one installation.
Basically, it's multiple DBs with different URIs inside one Neo4j DBMS installation.  
2. Another option is to have completely different clusters for each DB. This enables us to have independant scaling and full isolation.
3. Also, Fabric is a way to store and retrieve data in multiple databases, whether they are on the same Neo4j DBMS or in multiple DBMSs:
>
a unified view of local and distributed data, accessible via a single client connection and user session  
increased scalability for read/write operations, data volume and concurrency  
predictable response time for queries executed during normal operations, a failover or other infrastructure changes  
High Availability and No Single Point of Failure for large data volume.  
Data Federation: the ability to access data available in distributed sources in the form of disjointed graphs.  
Data Sharding: the ability to access data available in distributed sources in the form of a common graph partitioned on multiple databases.  


### Security
Neo4j supprots custom auth providers (eg OpenID connect provider), which allows us to integrate frontend with DB directly.  
Also, Neo4j supports fine-grained access control, so we can manage access for user role/specific user to Graph, sub-graph, Operation(read write property/relation) etc  
__This RBAC options can be used in multi tenancy implementations__


### side notes
Clustering, RBAC and other things are available only for Neo4j Enterprise Edition (paid)
Suggested storage type is low latency SSD
Suggested configuration of k8s AntiAffinity rules for core nodes to increase cluster stability


### resources
Official documentation is pretty good
https://neo4j.com/docs/operations-manual/current/clustering/introduction/
https://neo4j.com/labs/neo4j-helm/1.0.0/operations/
https://neo4j.com/docs/operations-manual/current/kubernetes/quickstart-cluster/server-setup/
https://neo4j.com/docs/operations-manual/current/fabric/introduction/
https://medium.com/neo4j/reactive-multi-tenancy-with-neo4j-4-0-and-sdn-rx-d8ae0754c35

