---
title: GraphQL Schema Design
author: Nick Harris
date: 2020-07-23 22:23:00 -0500
tags: [graphql]
---

![Desktop View]({{ "/assets/img/graphql-design/banner.jpg" | relative_url }})

# Introduction

[GraphQL](https://graphql.org/learn/) enables us to _explicitly_ describe our API to remove ambiguity from how our systems communicate. However, without a standard way of designing our API we can still find ourselves with an _explicitly unclear_ API.

With RESTful APIs the convention is to treat paths like resources with each pathname taking you deeper into a more specific resource of your data. GraphQL doesn't have these long accepted conventions to help guide us as to how we should design our Schema Graph. This means each team must explicitly define their own. At CyberGRX, we've discover these design patterns over the years either through hard learned lessons or heeding the warnings of others in the community.

# Versioning

The GraphQL spec recommends a [versionless](https://graphql.org/learn/best-practices/#versioning) design strategy which means changes are always backwards compatible. GraphQL supports this by enabling deprecation metadata for fields. This can be frustrating for some designers who want to keep a clean schema but it enables rapid development through asynchronous schema syncs between server and client. The client can wait to pull in the latest schema until they are ready for the changes.

With all of this said, if your GraphQL schema is private and for internal use only then you have a lot more control over coordinated breaking changes. However, this should be discouraged and reserved for only rare exceptions such as major redesign efforts.

> Don't add something to your schema until you need to expose that data. Adding data is cheap, taking it away is expensive. [YAGNI Principle](https://en.wikipedia.org/wiki/You_aren%27t_gonna_need_it)

# Queries

Treating paths as resources is actually a pretty good approach even for GraphQL. We can think of our first entry Queries like our first pathname of a URL. It represents a resource that gives context to other fields the deeper you go in your query. For example:

```graphql
query {
  myCompany {
    name
    owner {
      firstName
      lastName
    }
  }
}
```

The above query gives you the context of company with the term `my`. The `User` field `owner` also has context from the parent resource `Company` to show what company that user is the owner of. We could have designed our schema to separate these queries into more explicit flat root queries like this.

```graphql
query {
  myCompany {
    name
  }
  myCompanyOwner {
    firstName
    lastName
  }
}
```

This gives us the same information only we are not taking full advantage of the graph nature of GraphQL. When designing our API we need to ask ourselves what should our root resource be that gives us context to the rest of our data needs. In the above example our root resource that provides the foundational context for the rest of the data is the requester's Company. From there we are able to traverse resource relationships to find out more specific information like the company's owner.

This is a very simple example and real data will be more complex and a lot harder to identify what the root resource is. Which is why you should ask yourself the following questions to help you make at least an OK design decision if not always the best design decision:

1. How does the client get my required arguments to my query? Can another resource provide those required arguments?
1. Does my query make sense outside of the context of a parent resource?
1. What are the performance considerations? Is the parent resource expensive to query? Does the client always need the parent resource when fetching my query?

Each of these questions gets you to think about where a given data resource should get resolved. Lets dive deeper into some key query scenarios.

## Leveraging Parent Resource Context

The first question you should always ask yourself is whether there is a parent resource required for context. The biggest design smell is when you have a query that requires id's for multiple resource types or requires the ID of a different resource type than what is returned. For example:

```graphql
query {
  companyOwner(companyId: 1) {
    firstName
    lastName
  }
}
```

This would be better designed with the Company type as a parent to the owner.

```graphql
query {
  company(id: 1) {
    name
    owner {
      firstName
      lastName
    }
  }
}
```

This has the advantage of enabling the client to also request information on the company itself instead of having to send one query for the company and another query for the company owner.

## Performance > Good Design

This philosophy is more of a guideline to help you make good decisions when all things are equal. Sometimes a good design isn't the only factor. Query performance will also impact our design because sometimes a specific resource is expensive to query. If the client ultimately doesn't care about the data from the parent resource then it doesn’t matter how critical the parent resource is to the context of what is being queried. We save ourselves the network bandwith to the client because of GraphQL but we still took the DB hit unnecessarily on the backend. For example:

```graphql
query {
  myCompany {
    products {
      owner {
        firstName
        lastName
      }
    }
  }
}
```

In this example we want to know the product owners for all the products for the requesting user's company. Lets pretend that the DB that stores product information is in a separate system and so the lookup for products from a company is expensive. We might be better off directly querying for the owner instead of trying to pull in all the company and product information just to give the client the user information.

```graphql
query {
  myProductOwners {
    firstName
    lastName
  }
}
```

Here we know exactly the resource that is requested and so we can optimize our query to only lookup the data the client cares about from the correct DB.

> An alternative approach here would be to keep the Schema design and modify the resolvers to introspect of the query fields such that myCompany resolver does not fetch the Company node since no fields were requested and products resolver prefetches the Users for each product returned by performing a single batch fetch of the list of Users. This would solve the performance problem while keeping the preferred schema design but at the expense of resolver code complexity.

## Connection Pattern

The Connection pattern was created by Facebook’s Relay team to define the data structure for a Paginated list. This pattern has gained wide adoption even by designers who don’t use the Relay technology. At CyberGRX we opted to use a modified version of this pattern that explicitly uses offsets instead of cursors. Regardless of how you do your indexing, the benefits of these patterns are the same. It defines a self contained structure that includes:

- Page metadata. next/prev page booleans and start/end indices for the page
- List of items in the page
- Ability to inject custom metadata related to the connection query

```graphql
type PageInfo {
  hasNextPage: Boolean!
  hasPrevPage: Boolean!
  startOffset: Int
  endOffset: Int
}

type CustomerEdge {
  node: Company!
  offset: Int!
}

type CustomerConnection {
  pageInfo: PageInfo!
  edges: [Edge!]!
  totalCount: Int!
}

type Company {
  id: ID!
  name: String!
}

type Query {
  myCustomers(offset: Int, limit: Int): CustomerConnection!
}

query {
  myCustomers(offset: 0, limit: 25) {
    totalCount
    pageInfo {
      hasNextPage
      hasPrevPage
      startOffset
      endOffset
    }
    edges {
      node {
        id
        name
      }
      offset
    }
  }
}
```

This example shows how we containerize all the relevant data needed for the client to create a paginated Table view for a list of Customers. But what would this look like if we just did a simple List of data? We would need to define separate queries to get the metadata for the paginated view. Maybe something like this:

```graphql
type Query {
  myCustomers(offset: Int, limit: Int): [Company!]!
  myCustomerTotalCounts: Int!
  myCustomerHasNext(offset: Int!): Int!
  myCustomerHasPrev(offset: Int!): Int!
}

query {
  myCustomers(offset: 0, limit: 25) {
    id
    name
  }
  myCustomerTotalCounts
  myCustomerHasNext(offset: 0, limit: 25)
  myCustomerHasPrev(offset: 0, limit: 25)
}
```

If we put aside the subjective opinions of collocation of data versus a flat query design, we can still see some critical issues with this design. To see it we have to look deeper at the backend resolver functions. The main issue at hand here is the fact that each query executes separate calls to the DB and can run into data inconsistencies. We could run into a situation where the total count shows we have more data compared to the page we are on but hasNext says there are no more pages. This could make it so the user is unable to click next page even though there are still more results. This is not a good user experience. Keeping all the metadata within the same root query transaction allows the resolver function to ensure consistency in the data.

# Mutations

GraphQL only has a single write protocol definition unlike RESTful’s PUT, POST, and PATCH. This means naming conventions are really important for a team to convey intentions. At CyberGRX, we follow these naming conventions:

## Create

Create mutations should follow the naming convention of create[TYPE_NAME] and should not take in an ID as an argument. The id field will be generated by the server.

```graphql
mutation {
  createUser(firstName: "Jane", lastName: "Doe") {
    id
    firstName
    lastName
    createdAt
    updatedAt
  }
}
```

## Update and Replace

GraphQL explicit schemas have encouraged a lot of server implementers to map GraphQL schemas to existing model definitions either via standard Class types or more advanced modeling definitions. Because of this some server implementations have struggled to distinguish between NULL values vs missing keys. This has made PATCH equivalent operations to be very challenging. Because of how error prone patch updates are it is recommended to only define replace update operations.

```graphql
mutation {
  updateUser(id: 1, firstName: "Jane", lastName: "Smith") {
    id
    firstName
    lastName
  }
}
```

```json
{
  "data": {
    "updateUser": {
      "id": 1,
      "fistName": "Jane",
      "lastName": "Smith"
    }
  }
}
```

Now what happens if our schema doesn’t make lastName required and we exclude that value? What should our expectation be about the returned data response? If we are implementing an update and replace strategy our expectations should be that we just set that missing property to NULL

```graphql
mutation {
  updateUser(id: 1, firstName: "Jane") {
    id
    firstName
    lastName
  }
}
```

```json
{
  "data": {
    "updateUser": {
      "id": 1,
      "fistName": "Jane",
      "lastName": null
    }
  }
}
```

Some fields, like lastName, you may not want to ever be null but some might be appropriate. With update and replace strategy you should explicitly define what arguments are NonNull and what can be Null. This will ensure violators will error on the client side and avoid an unnecessary round trip to the server.

## Patch Update

All hope isn’t lost for patch updates. We just have to be very explicit in our schema definition to what we are updating. This is where naming conventions come in to save the day. If we don’t want the network bandwidth overhead of passing the entire user object for every update we can define explicit partial update mutations. This is obviously not ideal but anyone who has developed an edit form can tell you horror stories about the struggles of tracking null, empty string, undefined, and dirty fields. And that is just on the client side! The server side isn’t much better. Interpreting null values vs missing keys can be error prone and the cause of many bugs. Not to mention the glaring issue of how do you define your Input args when a field is required in the DB but optional to set for the mutation? GraphQL only allows you to define that a field is Nullable not whether it can be Undefined. This design takes the guess work out of GraphQL mutation contracts.

```graphql
mutation {
  updateUserLastName(id: 1, lastName: "Smith") {
    id
    firstName
    lastName
    createdAt
    updatedAt
  }
}
```

## Delete

Delete operations should be named delete[TYPE_NAME] and should return the deleted object. Many GraphQL articles will give delete examples that have an `ok: Boolean` response object. This is not recommended because of how Apollo caching works. Returning a handler to the modified object is always required so that the client cache can get appropriately updated. Client developers have options to ensure the cache gets correctly updated but we shouldn’t make it hard on them, so return the resource object.

```graphql
mutation {
  deleteUser(id: 1) {
    id
    firstName
    lastName
    createdAt
    updatedAt
  }
}
```

## Remove

While delete removes a resource object from the database, remove[LIST_ITEM] should remove an item from a logical list and not delete the resource itself from the database. This is a subjective naming convention that has no technical justification for adoption other than consistency which gives us self documenting schemas.

```graphql
mutation {
  removeCustomer(id: 1) {
    id
  }
}
```

In this example the logical list is implied because of our domain model. A company only has a single list of customers and so the root query does not need to provide more context to the server. But what if this was called deleteCustomer instead? If you remove one of your Customers does it delete them from the platform for other users? If the platform’s action button said Delete that would be a legitimate point of confusion for the user which means it would be a legitimate point of confusion for our Schema design too.

# Errors and Alternative Responses

This has been a contentious topic of debate within the GraphQL community since the beginning but the dust has settled and the majority of the community has come to an agreement. Schema designers should distinguish between an Error and an Alternative Response data structure. The main distinguishing difference between the two is:

Error - The operation failed and there is nothing the client can do about it. Someone should probably file a bug report

Alternative Response - An expected error happened and the client should be explicitly told what happened so they can choose to do something about it.

What this means is some “errors” are actually just one of the known workflows that the client should manage with an explicit user experience. The way you can design for this in GraphQL is with Union types. Lets look at a complex scenario where we have a long running operation to calculate risk scores that requires authorization by the third party and requires premium subscription with us. We want to succinctly support giving the client the right data so they can render the correct view. We can accomplish this with a Union of 4 response types

```graphql
enum AuthStatus {
  DENIED
  REQUESTED
}

type NotAuthorizedResponse {
  status: AuthStatus!
  companyRepEmail: String!
  deniedReason: String
}

type ScoresCalculatingResponse {
  progress: Float
  startedAt: Date!
}

type PremiumFeatureResponse {
  featureName: String!
  requiredPackage: String!
  salesRepEmail: String!
}

type Score {
  javelin: Int
  threatIntel: Int
}

union ScoreResponse =
    Score
  | PremiumFeatureResponse
  | ScoresCalculatingResponse
  | NotAuthorizedResponse

type ThirdParty {
  name: String!
  inheritRisk: String
  score: ScoreResponse
}

query {
  myThirdParties {
    name
    inheritRisk
    score {
      ... on Score {
        javelin
        threatIntel
      }
      ... on PremiumFeatureResponse {
        featureName
        requiredPackage
        salesRepEmail
      }
      ... on ScoresCalculatingResponse {
        progress
        startedAt
      }
      ... on NotAuthorizedResponse {
        status
        companyRepEmail
        deniedReason
      }
    }
  }
}
```

In this example we now have a very clean way for the client to render appropriate views for the alternative responses for getting a third party’s score. Because the score is NonNull this also means the client could choose to not care about one of the alternative responses

Lets look at a scenario where the client doesn’t care about upselling they only care about scores and in progress calculations. We don’t have to change the schema for this, the client can just ask for only what they care about.

```graphql
query {
  myThirdParties {
    name
    inheritRisk
    score {
      ... on Score {
        javelin
        threatIntel
      }
      ... on ScoresCalculatingResponse {
        progress
        startedAt
      }
    }
  }
}
```

```json
{
  "data": {
    "myThirdParties": [
      {
        "name": "Sample Company",
        "inheritRisk": "Foobar",
        "score": null // NotAuthorized or PremiumFeatureResponse
      }
    ]
  }
}
```

The client can choose in this scenario not render the score component if null and hide the fact that the user was not authorized to view scores for whatever reason. The schema design gives the client the flexibility to choose the user experience. But what if we raised a GraphQL error for this unauthorized data request and didn’t use Unions at all?

```graphql
query {
  myThirdParties {
    name
    inheritRisk
    score {
      javelin
      threatIntel
    }
  }
}
```

```json
{
  "data": {
    "myThirdParties": [
      {
        "name": "Sample Company",
        "inheritRisk": "Foobar",
        "score": null // NotAuthorized or PremiumFeatureResponse
      }
    ]
  },
  "errors": [
    {
      "message": "Not Authorized",
      "locations": [{ "lines": 1, "column": 17 }]
    }
  ]
}
```

In this schema design the client is now forced to only support not rendering scores when null. Alternative user experiences is not supported. The client has no idea whether the scores aren’t available yet, the third party didn’t authorize them, or if they haven’t paid for the correct premium access. In fact the user doesn’t even know scores could be presented on the view the client renders. This schema design forces a specific UX that may not be desirable and may potentially result in an error alert depending on how the client handles GraphQL Errors. Understanding the alternative responses is critical in schema design so that the end user experience is not restricted
