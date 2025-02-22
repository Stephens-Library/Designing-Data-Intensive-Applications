# Transaction Processing or Analytics?
In the early days of business data processing, a write to the database typically corresponded to a *commercial transaction* taking place: making a sale, placing an order with a supplier, paying an employee's salary, etc

As databases expanded into areas that didn't involve money changing hands, the term *transaction* nevertheless stuck, referring to a group of reads and writes that form a logical unit

*A transaction does not necessarily have ACID (atomicity, consistency, isolation, and durability) properties, transaction processing just means allowing clients ot make low-latency reads and writes, as opposed to batch processing jobs, which only run periodically*

ACID is are a set of properties that ensure reliable processing of database transaction:
- Atomicity: A transaction is treated as a single, indivisible unit, this means that all operations within the transaction must be completed successfully, if any part fails, the entire transaction is rolled back leaving the database unchanged
- Consistency: A transaction must transform the database from one valid state to another, preserving all predefined rules, constraints, and invariants, any transaction that violates these rules will not be allowed, ensuring data integrity
- Isolation: Even when multiple transactions are executed concurrently, each should operate as if it were the only one in the system, isolation prevents transactions from interfering with one another
- Durability: Once a transaction has been committed, its changes are permanent even in the event of system failure, this means the database will retain changes after a crash or power loss thanks to logging or redundant storage

Even though databases started being used for many different kinds of data, the basic access pattern remained similar to processing business transactions

An application typically looks up a small number of records by some key using an index, records are inserted or updated based on the user's input, because these applications are interactive, the access pattern became known as *online transaction processing* (OLTP)

Databases also started being used for *data analytics*, which has a very different access pattern, usually an analytic query needs to scan over a huge number of records, only reading a few columns per record, and calculates aggregate statistics (such as count, sum, or average) rather than returning the raw data ot the user

These queries are often written by business analysts and feed into reports that help the management of a company make better decisions

In order to differentiate this pattern of using databases from transaction processing, it has been called *online analytic processing*, the difference between OLTP and OLAP is not always clear-cut, but some typical characteristics are listed in the table below

![Image](<photos/OLTP_vs_OLAP.png>)

At first the same databases were used for both transaction processing and analytic queries, SQL turned out to be quite flexible in this regard

Nevertheless, in the late 1980s and early 1990s, there was a trend for companies to stop using their OLTP systems for analytic purposes and to run the analytics on a separate database instead, this separate database was called a *data warehouse*

## Data Warehousing
An enterprise may have dozens of different transaction processing systems: systems powering the customer-facing website, controlling point of sale systems in physical stores, tracking inventory in warehouses, planning routes for vehicles, managing suppliers, administering employees, etc

Each of these systems is complex and needs a team of people to maintain it, so the systems end up operating mostly autonomously from each other

These OLTP systems are usually expected to be highly available and to process transactions with low latency, since they are often critical to the operation of the business

Database administrators therefore closely guard their OLTP databases, they are usually reluctant to let business analysts to run ad hoc analytic queries on OLTP database, since those queries are often expensive, scanning large part of the dataset, which can harm the performance of concurrently executing transactions

A *data warehouse*, by contrast is a separate database that analysts cna query to their hearts' content, without affecting OLTP operations

The data warehouse contains a read-only copy of the data in all the various OLTP systems in the company

Data is extracted from OLTP databases (either a periodic data dump or a continuous stream of updates), transformed into an analysis-friendly schema, cleaned up, and then loaded into the data warehouse

This process of getting data into the warehouse is known as *Extract-Transform-Load* (ETL) and is illustrated as below

![Image](<photos/extract_transform_load.png>)

Data warehouses now exist in almost all large enterprises, but in small companies they are almost unheard of

This is probably because most small companies don't have so many different OLTP systems, and most small companies have a small amount of data, small enough that it can be queried in a conventional SQL database, or even analyzed in a spreadsheet

In a large company, a lot of heavy lifting is required to do something that is simple in a small company

A big advantage of using a separate data warehouse rather than querying OLTP systems directly for analytics is that the data warehouse can be optimized for analytic access patterns

It turns out that the indexing algorithms discussed in the first half of this chapter work well for OLTP, but are not very good at answering analytic queries, we will look at storage engines optimized for analytics instead

### The Divergence Between OLTP databases and Data Warehouses
The data model of a data warehouse is most commonly relational, because SQL is generally a good fit for analytic queries

There are many graphical data analysis tools that generate SQL queries, visualize the results, and allow analysts to explore the data (through operations such as *drill-down* and *slicing and dicing*)

On the surface, a data warehouse and a relational OLTP database look similar, they both have a SQL query interface, however, the internals of the systems can look quite different, because they are optimized for very different query patterns

Many database vendors now focus on supporting either transaction processing or analytic workloads but not both

Some databases, such as Microsoft SQL Server and SAP HANA, have support for transaction processing and data warehousing in the same product

However, they are increasingly becoming two separate storage and query engines, which happen to be accessible through a common SQL interface

Data warehouse vendors such as Teradata, Vertica, SAP HANA, and ParAccel typically sell their systems under expensive commercial licenses, Amazon RedShift is a hosted version of ParAccel

More recently, a plethora of open source SQL-on-Hadoop projects have emerged, they are young but aiming to compete with commercial data warehouse systems, these include Apache Hive, Spark SQL, Cloudera, Impala, Facebook Presto, Apache Tajo, and Apache Drill, some of them are based on ideas from Google's Dremel

## Stars and Snowflakes: Scehams for Analytics
A wide range of data models are used in the realm of transaction processing depending on the needs of the application

On the other hand, in analytics, there is much less diversity of data models, many data warehouses are used in a fairly formulaic style known as *star schema* (also known as *dimensional modeling*)

Below illustrates a data warehouse that may be found in a grocery retailer, at the center of the schema is a *fact table*, each fow of the fact table represents an event that occurred at a particular time, if we were analyzing website traffic rather than retail sales, each row m ight represent a page view or a click by a user

![Image](<photos/star_schema.png>)
*This figure shows an example of a star schema for use in a data warehouse*

Usually, facts are captured as individual events, because this allows maximum flexibility for analysis later

However, this means that the fact table can become extremely large, a big enterprise like Apple, Walmart, or eBay may have tens of petabytes of transaction history in its data warehouse, most of which is in fact tables

Some of the columns in the fact tables are attributes, such as the price at which the product was sold and the cost of buying it from the supplier (to calculate profit margin)

Other columns are foreign keys that reference to other tables called *dimension tables* as each row in the fact table represents an event, the dimension represents the *who, what, where, when, how, and why* of the event

Even date and time are often represented using dimension tables, because this allows additional information about dates to be encoded, allowing queries to differential between sales on holidays and non-holidays

The name "star schema" comes from the fact that when the table relationships are visualized, the fact table is in the middle, surrounded by its dimension tables; the connections to these tables are like rays of a star

A variation of this template is known as the *snowflake schema* where dimensions are further broken down into sub-dimensions

For example, there could be separate tables for brands and product categories, and each row in the `dim_product` table could reference the brand and category as foreign keys rather then storing them as strings in the `dim_product`

Snowflake schemas are more normalized then star schemas, but star schemas are often preferred because they are simpler for analysts to work with

In a typical data warehouse, tables are often very wide: fact tables often have over 100 columns, sometimes several hundred

Dimension tables can also be very wide, as they include all the metadata that may be relevant for analysis, for example the `dim_store` table may include details of which services are offered at each store, whether it has an in-store bakery, the square footage, the date when the store was first opened, when it was last remodeled, how far it is from the nearest highway, etc