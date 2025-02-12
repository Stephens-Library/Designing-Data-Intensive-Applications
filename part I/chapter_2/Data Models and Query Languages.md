# Data Models and Query Languages
Data models are perhaps the most important part of developing software, because they have such a profound effect: not only on how the software is written, but also on how we *think about the problem* we are solving.

Most applications are built by layering one data model on top of another. For each layer, the key question is: how is it *represented* in terms of the next-lowest layer? For example:
1. As an application developer, you look at the real world and model it in terms of objects, data structures, and APIs that manipulate those data structures, those structures are often specific to your application
2. When you want to store those data structures, you express them in terms of a general-purpose data model, such as JSON or XML documents, tables in a relational database or graph model
3. The engineers who built your database software decided on a way of presenting that JSON/XML/relational/graph data in terms of bytes in memory, on disk, or on a network. The representation may allow the data to be queried, searched, manipulated, and processed in various ways
4. On yet lower levels, hardware engineers have figured out how to represent bytes in terms of electrical currents, pulses of light, magnetic fields, and more

In a complex application there may be more intermediary levels, such as APIs built upon APIs, but the basic idea is still the same: each layer hides the complexity of the layers below it by providing a clean data model, these abstractions allow different groups of people to work together effectively

There are different kinds of data models, and every data model embodies assumptions about how it is going to be used, some kinds of usage are easy and some are not supported

Some operations are fast and some perform badly; some data transformations feel natural and some are awkward, it can take a lot of effort to master just one data model

Building software is hard enough, even when working with just one data model and without worrying about its inner workings, but since the data model has such a profound effect on what the software above it can and can't do, it's important to choose one that is appropriate to the application

We will look at a range of general-purpose data models for data storage and querying, in particular we will compare the relational data model, the document model, and a few graph based data models, in the next chapter we will discuss how storage engines work; that is, how these data models are actually implemented

## Relational Model Versus Document Model
The best-known data model today is probably that of SQL, based on the relational model proposed by Edgar Codd in 1970: data is organized into *relations* (called *tables* in SQL) where each relation is an unordered collection of *tuples* (*rows* in SQL)

The relational model was a theoretical proposal, and many people at the time doubted whether it could be implemented efficiently

However, by the mid-1980s, relational database management systems (RDBMSes) and SQL had become the tools of choice for most people who needed to store and query data with some kind of regular structure

The roots of relational databases lie in *business data processing*, which was performed on mainframe computers in the 1960s and '70s, the use cases appear mundane from today's perspective: typically *transaction processing* and *batch processing* (customer invoicing, payroll, reporting)

Other databases at that time forced application developers to think a lot about the internal representation of the data in the database, the goal of the relational model was to hide that implementation behind a cleaner interface

Over the years, there have been many competing approaches to data storage and querying, in the 1970s and early 1980s, the *network model* and the *hierachical model* were the main alternatives but the relational model came to dominate them

Object databases came and went in the late 1980s and early 1990s, XML databases appeared in the early 2000s, but have only seen niche adoption, each competitor to the relational model generated a lot of hype in its time, but it never lasted

As computers became vastly more powerful and networked, they started being used for increasingly diverse purposes

And remarkably, relational databases turned out to generalize very well, beyond their original scope of business data processing, much it be online publishing, discussion, and social networking, e-commerce, games, and SAAS applications

## The Birth of NoSQL
Now, in the 2010s, NoSQL is the latest attempt to overthrow the relational model's dominance, the name "NoSQL" was originally a catchy Twitter hashtag, and has been retroactively reinterpreted as *Not Only SQL*

There are several driving forces behind the adoption of NoSQL databases, including:
- A need for greater scalability than relational databases can easily achieve, including very large databases or very high write throughput
- A widespread preference for free and open source software over commercial database products
- Specialized query operations that are not well supported by the relational model
- Frustration with the restrictiveness of relational schemas, and a desire for a more dynamic and expressive data model

Different applications have different requirements, and the best choice of technology for one use case may well be different from the best choice for another use case

It therefore seems like that in the foreseeable future, relational databases will continue to be used alongside a broad variety of nonrelational datastores, an idea that is sometimes called *polygot persistence*

## The Object-Relational Mismatch
Most application development today is done in OOP languages, which leads to a common criticism of the SQL model, if the data is stored in relational tables, an awkward translation layer is required between the objects in the application code and the database model, this disconnect between the models is sometimes called *impedance mismatch*

Object-relational mapping (ORM) frameworks like ActiveRecord and Hibernate reduce the amount of boilerplate code required for this translation layer, but they can't completely hid the differences between the two models

For example, we will try to illustrate how a resume could be expressed in a relational schema. The first profile as a whole can be identified by a unique identifier, `user_id`. Fields like `first_name` and `last_name` appear exactly once per user, so they can be modeled as columns on the `users` table

However, most people have had more than one job in their career, and people may have varying number of pieces of contact information

There is a one-to-many relationship from the users to these items, which can be represented various ways:
- In the traditional SQL model, the most common normalized representation is to put positions, education, and contact information in separate tables, with a foreign key reference to the `users` table
- Later versions of the SQL standard added support for structured data types and XML data, this allowed multi-valued data to be stored within a single row, with support for querying and indexing inside those documents, these features are supported to varying degrees by Oracle, IBM, DB2, MS SQL Server, and PostgreSQL. A JSON datatype is also supported by several databases including IBM DB2, MySQL, and PostgreSQL
- A third option is to encode jobs, education, and contact info as a JSON or XML document, store it on a text 

For a data structure like a resume, which is mostly a self-contained `document`, a JSON representation can be quite appropriate , JSON has the appeal of being much simpler than XML, JSON has the appeal of being much simpler than XML, document-oriented databases like MongoDB, RethinkDB, CouchDB, and Espresso support this data model

```json
{
    "user_id": 251,
    "first_name": "Bill",
    "last_name": "Gates",
    "summary": "Co-chair of the Bill & Melinda  Gates... Active blogger.",
    "region_id": "us:91",
    "industry_id": 131,
    "photo_url": "/p/7/000/253/05b/308dd6e.jpg",
    "positions": [
        {"job_title": "Co-chair", "organization": "Bill & Melinda Gates Foundation"},
        {"job_title": "Co-founder, Chairman", "organization": "Microsoft"}
    ],
    "education": [
        {"school_name": "Harvard University", "start": 1973, "end": 1975},
        {"school_name": "Lakeside School, Seattle", "start": null, "end": null}
    ],
    "contact_info": {
        "blog": "http://thegatesnotes.com",
        "twitter": "http://twitter.com/BillGates"
    }
}
```

Some developers feel that the JSON model reduces the impedance mismatch between the application code and the storage layer

However, as we shall see in Chapter 4 (Encoding and Evolution) there are also problems with JSON as a data encoding format

The lack of a schema is often cited as an a