---
title: GraphQL Schema Design
author: Nick Harris
date: 2020-07-26 20:46:00 -0500
tags: [graphql]
summary: GraphQL enables us to explicitly describe our API to remove ambiguity from how our systems communicate; however, without a standard way of designing our API we can still find ourselves with an explicitly unclear API.
image: /assets/img/graphql-design/banner.jpg
---

# Introduction

[GraphQL](https://graphql.org/learn/) enables us to _explicitly_ describe our API to remove ambiguity from how our systems communicate. However, without a standard way of designing our API we can still find ourselves with an _explicitly unclear_ API.

With RESTful APIs, the convention is to treat paths like resources with each pathname taking you deeper into a more specific resource of your data. It also has documentation around how HTTP verbs interact with resources. GraphQL doesn't have these long accepted conventions to help guide our Schema design. This means each team must explicitly define their own conventions. These are some of the design strategies that I've found the most helpful.

# Versioning

The GraphQL spec recommends a [versionless](https://graphql.org/learn/best-practices/#versioning) design strategy which means changes are always backwards compatible. GraphQL supports this by enabling deprecation metadata for fields. This can be frustrating for some designers who want to keep a clean schema but it enables rapid development through asynchronous schema syncs between server and client. The client can wait to pull in the latest schema until they are ready for the changes.

> Don't add something to your schema until you need to expose that data. Adding data is cheap, taking it away is expensive. [YAGNI Principle](https://en.wikipedia.org/wiki/You_aren%27t_gonna_need_it)

# Queries

Treating paths as resources is actually a pretty good approach even for GraphQL. We can think of our first entry Queries like our first pathname of a URL. It represents a resource that gives context to other fields the deeper you go in your query. For example:

```graphql
query {
  myCompany {
    owner {
      name
    }
  }
}
```

Alternatively, we could have designed our schema to separate these queries into more explicit flat root queries like this.

```graphql
query {
  myCompany {
    name
  }
  myCompanyOwner {
    name
  }
}
```

This gives us the same information but is one better than the other? The answer is, it depends. Consider the following when designing your schema:

1. Favor less root queries
1. Leverage parent context data to reduce required query arguments
1. Let business workflow entry points define your root queries
1. Your GraphQL schema is not 1:1 to your Database Schema
1. Measure performance and optimize your schema based on common queries.

## Leveraging Parent Resource Context

Some domain data naturally has a parent context. The data just doesn't make sense outside of that context. The biggest design smell is when you have a query that requires id's for multiple resource types or requires the ID of a different resource type than what is returned. For example:

```graphql
query {
  companyOwner(companyId: 1) {
    firstName
    lastName
  }
}
```

Asking for the `ID` of a different resource type shows that the root context is something else. In this case, the `Company` is the root context. This would be better designed with the `Company` type as a parent to the `owner`.

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

While nesting data into ownership hierarchies may make logical sense for your data, it could cause some serious performance challenges. If the client doesn't request the parent data, GraphQL will save us the bandwidth to the client but our parent resolver still executed the database query. For example:

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

In this example we want to know the product owners for all the products for the requesting user's company. Lets pretend that the DB that stores product information is in a separate system and so the lookup for products is expensive. We might be better off directly querying for the owner instead of trying to pull in all the company and product information just to give the client the user information.

```graphql
query {
  myProductOwners {
    firstName
    lastName
  }
}
```

Here we know exactly the resource that is requested and so we can optimize our query to only lookup the data the client cares about from the correct DB.

> An alternative approach here would be to keep the Schema design and modify the resolvers to introspect of the query fields and only query the DB for the data requested. GraphQL provides this metadata in the `info` argument to the resolver.

## Connection Pattern

The [Connection](https://relay.dev/graphql/connections.htm#) pattern was created by Facebook’s Relay team to define the data structure for a Paginated list. This pattern has gained wide adoption even by designers who don’t use the Relay technology. A Connection defines a self contained structure that includes:

- Page metadata. next/prev page booleans and start/end indices for the page
- List of items in the page
- Ability to inject custom metadata related to the connection query

Here is a simple example of defining an offset based Connection for the `myCustomers` query.

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

So what does offset give you vs cursor? The biggest one is client side index math. If you want to jump to Page 5 when you are on Page 1, using offset allows the client to jump ahead using the offset and page size. Cursors require sequential flow through the data.

# Mutations

GraphQL only has a single write protocol called `Mutation` unlike RESTful’s `PUT`, `POST`, `PATCH` and `DELETE`. This means naming conventions are really important for a team to convey intentions. The following is how I've broken down Mutation field names to convey a consistent API for the different write strategies you may want to expose.

## Create

Create mutations should follow the naming convention of `create[TYPE_NAME]` and should not take in an `ID` as an argument. The `id` field will be generated by the server.

```graphql
mutation {
  createUser(firstName: "Jane", lastName: "Doe") {
    id # server generated
    firstName
    lastName
    createdAt # server generated
    updatedAt # server generated
  }
}
```

## Update

Updates to existing data generally fall into two categories replace the entire document object or partially update a subset of fields for the document object.

### Replace Entire Object

For updating the entire object we use the naming convention of, `update[TYPE_NAME]`. The input arguments should include all mutable fields for the object and required arguments should be marked as `NonNull`. The return type should be the type being modified. This is important so the client cache can get updated appropriately.

```graphql
type User {
  name: String!
  bio: String
}

type Mutation {
  updateUser(id: ID!, name: String!, bio: String): User
}

mutation {
  updateUser(id: 1, name: "Jane Smith", bio: "CEO of Awesomeness") {
    id
    name
    bio
  }
}
```

```json
{
  "data": {
    "updateUser": {
      "id": 1,
      "name": "Jane Smith",
      "bio": "CEO of Awesomeness"
    }
  }
}
```

Now what happens if we don't set `bio`? Our expectation should be that `bio` is set to `NULL`.

```graphql
mutation {
  updateUser(id: 1, name: "Jane Smith") {
    id
    name
    bio
  }
}
```

```json
{
  "data": {
    "updateUser": {
      "id": 1,
      "name": "Jane Smith",
      "bio": null
    }
  }
}
```

With update and replace the API schema will match the data validation. This leaves little room for confusion and bugs but at the expense of managing more data than the client needs. This strategy makes the most sense for forms that provide an ability to change multiple document fields with a single submit.

### Custom Field Setters

Even in REST there is an education effort with engineers to ensure the verbs `PUT`, `POST` and `PATCH` are used correctly. And even in REST partial updates are the source of developer mistakes. The core of the issue centers around distinguishing between `NULL` and `UNDEFINED`. Not to mention the confusion between required fields for the API vs required fields for the data. Throw in some complex cascading field validation scenarios and now your partial update gets really complicated because you need the entire model to properly validate updating a field.

The solution is to instead of supporting generic `patch[TYPE_NAME]` mutation functions, we create custom update functions for the needs of the client. This stays within our design goal of validation consistency between our API schema and our data while also enabling the client to perform partial updates.

```graphql
type Mutation {
  updateUserName(id: ID!, name: String!): User
}

mutation {
  updateUserName(id: 1, name: "Janet Smith") {
    id
    name
    bio
  }
}
```

```json
{
  "data": {
    "updateUserName": {
      "id": 1,
      "name": "Janet Smith",
      "bio": "CEO of Awesomeness"
    }
  }
}
```

This explicit mutation immediately clears up any ambiguity over what fields are required and what fields will get updated. I am now able to use GraphQL spec to enforce that the `id` and `name` are required fields and I will not be updating the optional `User` field `bio`. Whereas with a generic patch mutation function the GraphQL schema gives no guidance to the client as to what arguments are Nullable because it can't distinguish between allowing the argument to be missing vs allowing the argument to be Nullable.

## Delete

Delete operations should be named `delete[TYPE_NAME]` and will remove that resource and all references to it.

```graphql
type Mutation {
  deleteUser(id: ID!): Boolean
}
```

## Remove

While delete removes a resource object from the database, `remove[LIST_ITEM]` should remove an item from a logical list and not delete the resource itself from the database. This is a subjective naming convention that has no technical justification for adoption other than consistency.

```graphql
mutation {
  removeCustomer(id: 1) {
    id
  }
}
```

In this example the logical list is implied because of our domain model. A company only has a single list of customers and so the root query does not need to provide more context to the server. But what if this was called `deleteCustomer` instead? If you remove one of your Customers does it delete the Company from the platform for other users? If the platform’s UI button said `Delete Company` that would be a legitimate point of confusion for the user which means it would be a legitimate point of confusion for our Schema design too.

# Errors and Alternative Results

This has been a contentious topic of debate within the GraphQL community since the beginning but the dust has settled and the majority of the community has come to an agreement. Schema designers should distinguish between an Error and an [Alternative Result](https://medium.com/@sachee/200-ok-error-handling-in-graphql-7ec869aec9bc) data structure. The main distinguishing difference between the two is:

- **Error**: The operation failed and there is nothing the client can do about it. Someone should probably file a bug report
- **Alternative Result**: An expected error happened and the client should be explicitly told what happened so they can choose to do something about it.

What this means is some “errors” are actually just one of the known workflows that the client should manage with an explicit user experience. The way you can design for this in GraphQL is with `Union` types. Lets look at a complex scenario where we have a long running operation to calculate Cyber Risk Scores for your Third Parties that requires authorization by the third party and requires premium subscription with [CyberGRX](https://www.cybergrx.com). We want to succinctly support giving the client UI the right data so they can render the correct view. We can accomplish this with a Union of 4 response types

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

type Risk {
  name: String!
  category: String!
  score: Float!
  mitigationStrategy: String!
}

type Score {
  summary: String!
  topRisks: [Risk!]!
}

union ScoreResponse =
    Score
  | PremiumFeatureResponse
  | ScoresCalculatingResponse
  | NotAuthorizedResponse

type ThirdParty {
  name: String!
  score: ScoreResponse
}

query {
  myThirdParties {
    name
    score {
      ... on Score {
        summary
        topRisks {
          name
          category
          score
          mitigationStrategy
        }
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

In this example we now have a very clean way for the client to render appropriate views for the alternative results for getting a third party’s score. Because the score is NonNull this also means the client could choose to not care about one of the alternative results

Lets look at a scenario where the client only cares about scores and in progress calculations. We don’t have to change the schema for this, the client can just ask for only what they care about.

```graphql
query {
  myThirdParties {
    name
    score {
      ... on Score {
        summary
        topRisks {
          name
          category
          score
          mitigationStrategy
        }
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
        "score": {} // NotAuthorized or PremiumFeatureResponse
      }
    ]
  }
}
```

The client can choose in this scenario to not render the score component if score is empty and hide the fact that the user was not authorized to view scores for whatever reason. The schema design gives the client the flexibility to choose the user experience. But what if we raised a GraphQL error for this unauthorized data request and didn’t use Unions at all?

```graphql
query {
  myThirdParties {
    name
    score {
      summary
      topRisks {
        name
        category
        score
        mitigationStrategy
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
        "score": null // NotAuthorized or PremiumFeatureResponse or InProgress
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

In this schema design the client is now forced to only support not rendering scores when null. Alternative user experiences is not supported. The client has no idea whether the scores aren’t available yet, the third party didn’t authorize them, or if they haven’t paid for the correct premium access. In fact the user doesn’t even know scores _could_ be presented on the view the client renders. This schema design forces a specific UX that may not be desirable and may potentially result in an error alert depending on how the client handles GraphQL Errors. Understanding the alternative results is critical in schema design so that the end user experience is not restricted
