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
![image](https://user-images.githubusercontent.com/1872337/149520470-050d5923-c93e-4cf4-b5a8-b23682bd1dc5.png) (not strictly bound to GCP)


Each node should have own persistent storage (Azure disk). Suggested storage type is _Premium_LRS_ at least. We create AzureDisks and then use it from k8s neo4j nodes through k8s PersistentVolumeClaims.    
The easiest way is to use helm v3 charts. 
There are helm charts to deploy core nodes(__chart neo4j/neo4j-cluster-core__), as well as needed amount of read replicas(__neo4j/neo4j-cluster-read-replica__). 

![image](https://user-images.githubusercontent.com/1872337/149521107-6b309911-6055-4d82-a462-6242ac0bb80b.png)
(example of server side routing used)  

Additionally, we might need to deploy __neo4j-cluster-headless-service__ to have an access from inside of k8s, and __neo4j/neo4j-cluster-loadbalancer__ to have an access from the outside of k8s cluster.  

k8s resources used: __StatefulSet__ for core and replica nodes, __ServiceAccount__ to link to cloud IAM, __Service__ as load balancer, __HPA__ to scale replica nodes etc  
  
More details:  
https://neo4j.com/docs/operations-manual/current/kubernetes/quickstart-cluster/access-outside-k8s/  
https://neo4j.com/docs/operations-manual/current/kubernetes/accessing-cluster/  

## Networking
Mainly, for external consumers, Neo4j cluster should expose 3 protocols - __Http:7474, https:7473 and bolt:7687__. Also, additional ports/protocls [are used under the hood (eg discovery, transactions, RAFT protocol etc](https://neo4j.com/docs/operations-manual/current/configuration/ports/)  

Accessing cluster wiht neo4j:// URI scheme enables driver to use server-side routing. If a Neo4j Driver connects to this cluster member, then the Neo4j Driver sends all requests to that cluster member, and then cluster member [can re-route each request in case of necessity](https://neo4j.com/docs/operations-manual/current/clustering/internals/).  

![image](https://user-images.githubusercontent.com/1872337/149797376-f954cc04-8dff-490e-b710-48466d2488a9.png)  

To expose those ports in AKS deployment, by default k8s __service with type LoadBalancer__ is used. Basically, __we need to create one static IP address per cluster__ like _az network public-ip create_ and assign it to this Load balancer.   
It's possible to use other network entitites, such as the __nginx-ingress__ controller, but they need to be configured to support TCP connections ([e g configuration for nginx](https://kubernetes.github.io/ingress-nginx/user-guide/exposing-tcp-udp-services/)). Also, additional configuration is required (Neo4j discovery addresses etc)  
![image](https://user-images.githubusercontent.com/1872337/149803522-4063fe23-e793-4f50-80d3-02de434aee5c.png)

Aditional info:  
https://community.neo4j.com/t/cannot-connect-to-cluster-using-k8s-ingress/15476  
https://github.com/neo4j-contrib/neo4j-helm/issues/42  
https://community.neo4j.com/t/expose-helm-installed-neo4jj-using-ingress-nginx/20056  



## Autoscaling
Scaling read replicas is relatively easy using HorizontalPodAutoscaler based on CPU load (needs minimum configuration)  
Scaling core members __is not recommended__ (as it involves rebalancing the cluster and replicating the data):
> A store copy is a major operation, which may disrupt the availability of instances in the cluster.  
> Automated scaling applies only to read replicas. At this time we do not recommend automatic scaling of core members of the cluster at all, and core member scaling should be limited to special operations such as rolling upgrades, documented separately. 

## Monitoring
Neo4j supports prometheus metrics(regular k8s flow), and Graphite and JMX - no significant problems expected  

## Multi tennancy options
1. Starting from version 4.0, Neo4j supports Multi Tenancy.
> we can create and use more than one active database at the same time. This works in standalone and causal cluster scenarios and allows us to maintain multiple, separate graphs in one installation.
Basically, it's multiple DBs with different URIs inside one Neo4j DBMS installation.  
2. Another option is to have completely different clusters for each DB. This enables us to have independant scaling and full isolation.
3. Fabric - basically, it's a meta DB that allows us to join multiple disjoint graphs. Has 3 possible deployment configurations based on requirements:
    - Development deployment (1 DBMS with data graphs and virtual meta graph)
    - Cluster deployment with no single point of failure (production with HA - at least 2 Fabric DBMS and cluster with at least 3 core members to host disjoint graphs)
    - Multi-cluster deployment (high scalability and availability with no single point of failure(miltiple Fabric instances, multiple data clusters)



## [Fabric](https://neo4j.com/docs/operations-manual/current/fabric/introduction/)
is a way to store and retrieve data in multiple databases, whether they are on the same Neo4j DBMS or in multiple DBMSs, using a single Cypher query. Fabric achieves a number of desirable objectives:

-   a unified view of local and distributed data, accessible via a single client connection and user session

-   increased scalability for read/write operations, data volume and concurrency

-   predictable response time for queries executed during normal operations, a failover or other infrastructure changes

-   High Availability and No Single Point of Failure for large data volume.

In practical terms, Fabric provides the infrastructure and tooling for:

-   **Data Federation**: the ability to access data available in distributed sources in the form of **disjointed graphs**.

-   **Data Sharding**: the ability to access data available in distributed sources in the form of a **common graph partitioned on multiple databases**.

With Fabric, a Cypher query can store and retrieve data in multiple federated and sharded graphs.


### Security
Neo4j supprots custom auth providers (eg OpenID connect provider), which allows us to integrate frontend with DB directly.  
Also, Neo4j supports fine-grained access control, so we can manage access for user role/specific user to Graph, sub-graph, Operation(read write property/relation) etc  
__This RBAC options can be used in multi tenancy implementations__


### Side notes
Clustering, RBAC and other things are available only for Neo4j Enterprise Edition (paid)
Suggested storage type is low latency SSD
Suggested configuration of k8s AntiAffinity rules for core nodes to increase cluster stability


### Resources
Official documentation is pretty good
https://neo4j.com/docs/operations-manual/current/clustering/introduction/
https://neo4j.com/labs/neo4j-helm/1.0.0/operations/
https://neo4j.com/docs/operations-manual/current/kubernetes/quickstart-cluster/server-setup/
https://neo4j.com/docs/operations-manual/current/fabric/introduction/
https://neo4j.com/docs/operations-manual/current/configuration/network-architecture/
https://github.com/neo4j-contrib/neo4j-helm/blob/master/templates/discovery-lb.yaml
https://neo4j.com/labs/neo4j-helm/1.0.0/externalexposure/
https://medium.com/neo4j/reactive-multi-tenancy-with-neo4j-4-0-and-sdn-rx-d8ae0754c35

