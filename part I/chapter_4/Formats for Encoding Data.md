# Introduction
Applications inevitably change over time, features are added or modified as new products launch, user requirements become better understood, or business circumstances change

In chapter 1, we introduced the idea of *evolvability*, we should aim to build systems that make it easy to adapt to change

In most cases, a change to an application's feature also requires a change to data that it stores, perhaps a new field or a record type needs to be captured or perhaps existing data needs to be presented in a new way

The data models we discussed have different ways of coping with such change, relational databases generally assume that all data in the database conforms to one schema: although that schema can be changed, there is exactly one schema in force at any point in time

By contrast, schemaless databases don't enforce a schema, so the database can contain a mixture of older and newer data formats written at different times

When a data format or schema changes, a corresponding change to application code often needs to happen, however ina  large application code changes cannot happen instantaneously

With server-side applications you may want to perform a *rolling upgrade* deploying the new version to a few nodes at a time, this allows new versions to be deployed without service downtime

With client-side applications you're at the mercy of the user who may not install the update for some time, this means that old and new versions of the code and old and new data formats may potentially all co-exist at the same time

In order for the system to continue running smoothly, we need to maintain compatibility in both directions:
- *Backward compatibility*: Newer code can read data written by older code
- *Format compatibility*: Older code can read data written by newer code

Backward compatibility is normally not hard to achieve, as author of the newer code you know the format of data written by older code, but forward compatibility can get tricky

We will look at several formats for encoding data, this includes JSON, XML, Protocol Buffers, Thrift, and Avro

In particular, we will look at how they handle schema changes and support old and new data, we will discuss how those formats are used for data storage and communication: in web services, Representational State Transfer (REST), remote procedure calls (RPC), and message-passing systems such as actors and message queue