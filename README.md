# coralql
A graphql that plays better with nature


This repo serves as a place to document my (and yours?) ideas and specifications for CoralQL. 


## REST

Pros:

* Seperate api's for each endpoint
* No specific data format (you can use json, messagepack, or custom)
* Easy to split up for microservices
* Can dynamically proxy/gateway easily

Cons:
 
* Paths have dynamic info in them (/users/128) which make it hard to filter logs/traffic, etc. by endpoint. Requires a lot of tooling.
* Noisy
* Often goes undocumented
* No one actually follows HATEOS
* No obvious one place to go for documentation
* hard to know what should be a path vs query vs body parameter

## GraphQL

Pros:

* A standard form of documentation
* self-documenting
* Single request for getting full graph of data

Cons:

* Every endpoint is the same url. Impossible to filter access logs on endpoint without reading the body (bad security practice)
* Documentation is actually really hard to use. You have to know what you are looking for, and requires a lot of back-and-forth. 
* Forces JSON
* Hard to dymically proxy/gateway
* Encourages monolithic structure
* N+1 SQL queries are easy to create and often require additional libraries and overhead to avoid.


## Pros of protobuf

* Strict typing
* Single url per endpoint. No dynamic routes.
* Self-documenting (you have to have the same proto file on the client and server)

Cons:
* client needs a compiled proto spec file. Discovery is not possible
* No documentation/discovery. You have to have a proto file.
* Hard to split up services because you need clients to update to two proto files
* impossible to proxy/gateway dynamically


# Time for something new

Taking these pros and cons, its time to make something new.
Essentially, this problem: ![xkcd standards comic](https://imgs.xkcd.com/comics/standards.png)

CoralQL attempts to be a new one by doing these things:

* Every endpoint has a unique static route `/user-profile` rather than dynamic `/users/5`. (protobuf)
* Every endpoint is easy to dynamically discover (graphql)
* Every endpoint is easy to proxy/gateway encouraging microservices (REST)
* Self-Documenting (GraphQL)
* Not tied to a message format (REST)


