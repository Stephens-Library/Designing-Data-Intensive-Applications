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