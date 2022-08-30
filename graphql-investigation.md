# Problem:
We have multiple data owners. Each data owner team has their own data source. 
Mainly, data source is different Cosmos DB instances, but some of them are REST API.

## We want to find a solution that could: 
1. Provide unified interface for queriyng data from different data sources
2. Do we need to have possiblity to merge entities in one query?
<details>
  <summary>Example of diffrent entities</summary>

  ```javascript
  // example - one entity for facade, but data loaded from different data sources
type Consumer {
    Id: String,
    Appointments: Appointment[], // loads from AppointmentService CosmosDB
    DigitalExperience: DigitalExperience[], // loads from analytics data source
    MedicationsHistory: MedicationsHistory[] // loads from MedicationsHistory API
}
  ```
  
</details>

3. Data schema should be easily changable by data owners
    - adding new data points into the data store
    - ensuring backward compatibility of changes(no field renaming/removing)
4. Data goverance
    - Security
        - Field based access rights mechanism?
        - Geolocation based filtering?
        - Who and how determines security rules?
        - PHI requirements are out of scope


## Things that GraphQL Gateway API should support:

1. Schema registry
    - *Reporting schema for consumers*
    - *Request validation*
2. Schema versioning
    - *Possbility to publish pre-release versions of the schema with changes*
3. Schema compilation
    - *Compile meta-schema using multiple sub-schemas*
    - *Schema stiching VS Schema federation*
        - https://netflixtechblog.com/how-netflix-scales-its-api-with-graphql-federation-part-1-ae3557c187e2
        - https://www.youtube.com/watch?v=Vq0ajno-zgw
4. Metrics 
    - performance reporting
    - historical stats 
    - metrics collection
    - request error tracking
    - schema field usage statistics (critical for schema evloution)
5. Security / access control
6. Caching
7. Rate limiting / Query Costing

## Process questions:
1. easiness of updating schema of each service
2. validation of changes againts backward compatibility(renaming/removing of one property in sub-graph could break the whole graph for some consumers)
3. who(what team) is responsible for query plans?

## Options considered:

* **Apollo studio**
    - Pros
        - schema registry and dynamic schema registration
        - schema versioning and different environments to test changes
        - metrics reporting
        - schema stiching and federation(not dynamic thou)
        - CLI tool for integration with CI/CD
        - gathering and displaying query plans
        - various team collaboration tools (UI, automated checks for schema changes etc)
    - Cons
        - SaaS, but enterprise  version has [Data Processing Agreement](https://www.apollographql.com/pricing/)
* **Azure Api Management** 
    - Pros
        - security
        - metrics
    - Cons
        - not a gateway server, more like a hosting platform 
        - basic GraphQL support, we need to implement all things manually
* **Apigee** 
    - Pros
        - security
        - metrics
    - Cons
        - not a gateway server, more like a hosting platform 
        - basic GraphQL support, we need to implement all things manually
* **AWS AppSync** 
    - Pros
        - more like set of custom resolvers for GraphQL
    - Cons
        - hard coupling with AWS infra
* **https://github.com/hasura/graphql-engine** 
    - Pros
        - security
        - schema merging (similar with shcema stiching, but restricted)
        - Admin UI
        - schema versioning and migrations
    - Cons
        - no support for cosmos db
        - very simplified metrics and other req.
* **[Prisma](https://github.com/prisma/prisma)**
     - just orm, not exactly what we need
* **[Hot chocolate](https://chillicream.com/docs/hotchocolate/distributed-schema/schema-federations)** 
    - dotnet implementation of the server, not an API gateway. Has remote schema possibility thou.
* **[Braid](https://bitbucket.org/atlassian/graphql-braid)** 
    - java implementation of library that can merge GraphQL schemas. Not exactly what we need.
* **https://github.com/pipedrive/graphql-schema-registry**
    - Pros
        - schema registry (dynamic and static)
        - schema versioning
        - field usage analyzer
        - based on Apollo server
        - Open source with active development, more features are in the roadmap
    - Cons
        - JS/TS based
        - error tracking and query costing should be implemeted manually for ex [with this library](https://github.com/pipedrive/graphql-query-cost)


