---
layout: post
title:  What is GraphQL
categories: [GraphQL, Backend Development]
tags: [GraphQL, Web API, REST vs GraphQL]
description: GraphQL lets clients request specific data efficiently, but has challenges with caching and performance.
---

It is one of several ways for clients and servers to communicate.
Although it's not as widely used as RESTful API, GraphQL is being used in various places due to its distinct advantages.

Let's first look at how GraphQL differs from REST API

Below are examples of REST API request methods. Generally, it uses GET, POST, PUT, DELETE methods.

```bash
# GET request example (Retrieve user information)
GET /api/users/123

# POST request example (Create a new user)
POST /api/users
{
	"name": "John Doe",
	"email": "john@example.com"
}

# PUT request example (Update user information)
PUT /api/users/123
{
	"name": "Jane Smith",
	"email": "jane@example.com"
}

# DELETE request example (Delete a user)
DELETE /api/users/123
```

On the contrary, GraphQL uses only the POST method for all requests.

```bash
# POST request example (Handle all operations with a single endpoint)

# Read user data ( GET )
POST /graphql
{
  "query": `
    query {
      user(id: 123) {
        name
        email
      }
    }
  `
}

# Create user mutation
POST /graphql
{
  "query": `
    mutation {
      createUser(
        name: "John Doe", 
        email: "john@example.com") 
        {
          id
          name
          email
        }
    }
  `
}

# Update user information mutation
POST /graphql
{
  "query": `
    mutation {
      updateUser(
        id: 123, 
        name: "Jane Smith", 
        email: "jane@example.com") 
        {
          id
          name
          email
        }
    }
  `
}

# Delete user mutation
POST /graphql
{
  "query": `
    mutation {
      deleteUser(id: 123) {
        success
      }
    }
  `
}
```

GraphQL sends requests as shown above, sending a query through a POST request in the requestBody called query, which includes both the request being sent and the desired response values.

```bash
{
  "query": `
    query {
      user(id: 123) {
        name
        email
      }
    }
  `
}
```
For example, the query above sends an id of 123 and requests the name and email of that user as the response value.

---

So what problems did GraphQL try to solve?

### 1. **Over-fetching: Retrieving too much data from the server**
An example of over-fetching is when retrieving user information, you also get data that isn't needed.
For instance, when the client only needs the current user's ID, but the REST API returns not only the ID but also the name, email, address, etc.
   
This results in the client receiving unnecessary data.
To solve this problem, it's important to only retrieve the data needed by the client.

At this point, it becomes possible through GraphQL to write the query exemplified above to retrieve only the needed ID.

### 2. **Under-fetching: Retrieving too little data from the server**
An example of under-fetching is when you want to retrieve a specific user from the user table and the name of the group that user belongs to from the group table.

In this case, with REST API, you need to send two requests.
(Of course, you can get everything with one request, but it's inefficient to write all the REST APIs one by one in large-scale projects.)

With GraphQL, you can retrieve all the desired data with a single request in such situations.

```bash
{
  "query": `
    query {
      user(id: 123) {
        id
        groups {
          name
        }
      }
    }
  `
}
```

In fact, in most cases like this, over-fetching and under-fetching often occur together.
Retrieving all user information (over-fetching because you're getting everything when you only need the ID) and all group information (over-fetching because you're getting everything when you only need the group name),
and because you can't get everything you want with each API, you have to request twice, resulting in under-fetching.

---

There are several types of query statements in GraphQL besides simple retrieval queries.

1. **Query**: Used to retrieve data.
2. **Mutation**: Used to create, update, and delete data.
3. **Subscription**: Used to receive real-time notifications when data changes.

We've already explained simple retrieval queries and Mutations, so let's look at Subscriptions.

**Subscriptions are used to receive real-time notifications when data changes.**
For example, in a chat application, you might receive real-time notifications when a new message arrives.
In such cases, REST API requires the client to periodically request data from the server, but with GraphQL, the server can send notifications to the client when data changes.

The method of implementing Subscription is roughly as follows:

1. Server-side implementation:
    - Define Subscription type in GraphQL schema.
    - Implement Subscription resolver.
    - Connect with the client using WebSocket or other real-time protocols.

2. Client-side implementation:
    - Write Subscription query.
    - Set up WebSocket connection.
    - Handle real-time updates received from the server.

---

Based on what we've learned so far, it seems to have many convenient aspects.
The client can retrieve only the data it wants, and the server doesn't need to create separate APIs for each client requirement.

However!

The fact that REST API is still popular means that GraphQL has disadvantages that make it not applicable everywhere.

### We can identify two main disadvantages of GraphQL:

**1. Caching is difficult**
   - It's difficult to cache because requests are written as complex queries that keep changing.

**2. Performance issues**
   - The process of interpreting complex and diverse queries coming as requests can burden the server.

---

### So, what kind of services are suitable for applying GraphQL?

**1. Services with complex and extensive data models**
   - Services where overfetching and underfetching frequently occur may find it more advantageous to use GraphQL in terms of server load and performance.

**2. Services where clients have a lot of control over data requests**
   - Services that require clients to send various forms of data requests can also provide more efficient service through GraphQL.

**3. Services with frequent data changes (services requiring real-time response)**
   - Services with frequent data changes can use Subscriptions to receive real-time updates.

---

### Use cases of major companies
- Facebook

Performance optimization in various devices and network environments
- Shopify

Providing customized data for various store themes and apps
- GitHub

Flexibly querying complex data structures (repositories, issues, pull requests, etc.)



[The link is a soccer team management program I created using GraphQL](https://github.com/wellsbabo/soccer-team-management-program-GraphQL)
