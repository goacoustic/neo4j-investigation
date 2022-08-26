
## Things that GraphQL engine should support:

1. Schema validation / registry 
2. security/access control
3. caching
4. performance reporting / historical 5. stats / metrics collection
6. Rate limiting / Query Costing
7. load measurement
8. error tracking
9. Schema evolution tooling / schema field usage statistics / field deprecations

## Process questions:
1. easiness of updating schema of each service
2. validation of changes againts backward compatibility(renaming/removing of one property in sub-graph could break the whole graph for some consumers)
3. who(what team) is responsible for query plans?

## Options considered:

* **Apollo** - SaaS, but enterprise  version has [Data Processing Agreement](https://www.apollographql.com/pricing/)
* **Azure Api Management** - basic graphql functionality, can handle security and some other parts, but server implementation lays on our shoulders
* **Apigee** - only basic graphql functionality
aws appsync - more like set of custom resolvers, tighly integrated with aws infra
* **https://github.com/hasura/graphql-engine** - good solution, but no support for cosmos db
* **[Prisma](https://github.com/prisma/prisma)** - just orm, not exactly feets
* **[Hot chocolate](https://chillicream.com/docs/hotchocolate/distributed-schema/schema-federations)** - dotnet implementation of the server, not gateway
* **[Braid](https://bitbucket.org/atlassian/graphql-braid)** - java implementation of library that can merge GraphQL schemas
* **https://github.com/pipedrive/graphql-schema-registry** - JS based open-source GraphQL gateway implemetation. Based on Apollo server, also supports error reporting and in the future usage analyzing 


