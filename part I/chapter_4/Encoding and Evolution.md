# Encoding and Evolution
Applications inevitably change over time, features are added or modified as new products are launched, user requirements become better understood, or business circumstances change

In most cases, a change to an application's features also requires a change to data that it stores: perhaps a new field or record type needs to be captured, or perhaps existing data needs to be presented in a new way

The data models we discussed have different ways of coping with such change, relational databases assume all data conforms to one schema (this means changing data through schema migrations such as the `ALTER` statement), on the other hand document models don't enforce a schema so the database can contain a mixture of older or newer data formats written at different times

When a data format or schema changes, a corresponding change to application code often needs to happen, however in a large application, code changes often cannot happen instantaneously

With server-side applications, you may want to perform a *rolling update* (also known as a *staged rollout*), deploying the new version to a few nodes at a time, checking whether the new version is running smoothly, and gradually working your way through all the nodes, this allows new versions to be deployed without service downtime, and thus encourages more frequent releases and better evolvability

With client-side applications you're at the mercy of the user, who may not install the update for some time

This means that old and new versions of the code, and old and new data formats, may potentially all coexist in the system at the same time, in order for the system to continues running smoothly, we need to maintain compatibility in both directions

**Backward Compatibility**: Newer code can read data that was written by older code
**Forward Compatibility**: Older code can read data that was written by newer code

Backward compatibility is normally not hard to achieve: as author of the newer code, you know the format of data written by older code, so you can explicitly handle it

Forward compatibility can be tricker, because it requires older code to ignore additions made by a newer version of code

In this chapter we will look at several formats for encoding data, including JSON, Protocol Buffers, Thrift, and Avro

In particular, we will look at how they handle schema changes and how they support systems where old and new data and code need to co-exist

We will then discuss how those formats are used for data storage and for communication: in web services, Representational State Transfer (REST), and remote procedure calls (RPC), as well as message-passing systems such as actors and message queues
