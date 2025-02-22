# Column-Oriented Storage
If you have trillions of rows and petabytes of data in your fact tables, storing and querying them efficiently becomes a challenging problem

Dimension tables are usually much smaller (millions of rows), so in this section, we will concentrate primarily on storage of facts

Although fact tables are often over 100 columns wide, a typical data warehouse query only accesses 4 or 5 of them at one time

Take this query below:
```sql
SELECT
    dim_date.weekday, dim_product.category,
    SUM(fact_sales.quantity) AS quantity_sold
FROM fact_sales
    JOIN dim_date ON fact_sales.date_key = dim_date.date_key
    JOIN dim_product ON fact_sales.product_sk = dim_product.product_sk
WHERE
    dim_date.year = 2013 AND
    dim_product.category IN ('Fresh fruit', 'Candy')
GROUP BY
    dim_date.weekday, dim_product.category;
```

This query only accesses three columns, but a bunch of rows (every occurrence of someone buying a fruit or candy during the 2013 calendar year)

How can we execute this query efficiently?

In most OLTP databases, storage is laid out in a *row-oriented* fashion: all the values from one row of a table are stored next to each other, document databases are similar: an entire document is typically stored as one contiguous sequence of bytes

In order to process a query, you may have indexes on `fact_sales.date_key` and/or `fact_sales.product_sk` that tell the storage engine where to find all sales for a particular date or for a particular product, but then, a oriented storage engine still needs to load all those rows from disk to memory, parse them, and filter out those that don't meet the conditions, this can take a long time

The idea behind *column-oriented storage* is simple: don't store all the values from one row together, but store all the values from each *column* together instead, if each column is stored in a separate file, a query only needs to read and parse those columns that are used in the query, which can save a lot of work

![Image](<photos/column-oriented_storage.png>)
*This figure depicts storing relational data by column, rather than by row

The column-oriented storage layout relies on each column file containing the rows in the same order, thus if you need to reassemble an entire row, you can take the 23rd entry from each of the individual column files and put them together to form the 23rd row of the table

## Column Compression
Besides only loading those columns from disk that are required for a query, we can further reduce the demands on disk throughput by compressing data

Fortunately, column-oriented storage often lends itself very well to compression

One technique that is particularly effective in data warehouses is *bitmap encoding*
![Image](<photos/bitmap_encoding.png>)

Often, the number of distinct values in a column is small compared to the number of rows (for example, a retailer may have billions of sales transactions, but only 100,000 distinct products)

We can now take a column with *n* distinct values and turn it into *n* separate bitmaps: one bitmap for each value with one bit for each row, the bit is 1 if the row has the value and 0 if not

If *n* is very small such as a `country` column which may have 200 distinct values, those bitmaps can be stored with one bit per row, but if *n* is bigger, there will be a lot of zeroes in most of the bitmaps (we say that they are *sparse*)

In that case, the bitmaps can additionally be run-length encoded, as shown at the bottom of the figure above, this can make the encoding more compact

Bitmap indexes such as these are very well suited for the kinds of queries that are common in a data warehouse, for example:
```sql
WHERE product_sk IN (30, 68, 69):
```
Load the three bitmaps for `product_sk = 30`, `product_sk  = 68` and `product_sk = 69`, and calculate the bitwise `OR` of the three bitmaps, which can be done very efficiently

On the other hand:
```sql
WHERE product_sk = 31 AND store_sk = 3:
```
Would calculate the bitwise `AND`, this works because the columns contain the rows in the same order, so the *k*th bit in one column's bitmap corresponds to the same row as the *k*th bit in another column's bitmap

### Column-Oriented Storage and Column Families
Cassandra and HBase have a concept of *column families*, which they inherit from Bigtable

However, it is very misleading to call them column-oriented: within each column family, they store all columns from a row together, along with a row key, and they do not use column compression, thus the Bigtable model is still mostly row-oriented

### Memory Bandwidth and Vectorized Processing
For data warehouse queries that need to scan over millions of rows, a big bottleneck is the bandwidth for getting data from disk into memory, however that is not the only bottleneck

Developers of analytical databases also worry about efficiently using the bandwidth from main memory into the CPU cache, avoiding branch mispredictions and bubbles in the CPU instruction processing pipeline, and making use of single-instruction-multi-data (SIMD) instructions in modern CPUs

Besides reducing the volume of data that needs to be loaded from disk, column-oriented storage layouts are also good for making efficient use of CPU cycles

For example, the query engine can take a chunk of compressed column data that fits comfortably in the CPU's L1 cache and iterate through it in a tight loop

A CPU can execute such a loop much faster than code that requires a lot of function calls and conditions for each record that is processed, column compression allows more rows from a column to fit in the same amount of L1 cache

Operators, such as bitwise `AND` and `OR` described previously, can be designed to operate on such chunks of compressed column data directly, this technique is known as *vectorized processing*

*Note*: SIMD (Single Instruction, Multi Data) is a type of parallel processing built into modern CPUs, it allows a single instruction to be executed on multiple data points simultaneously, rather than processing each piece at a time

A single piece of instruction issued by the CPU operates on all the elements on a vector register concurrently

Vector processing loads the data elements into vector registers quickly by loading the data into blocks (vectors)

"Bubbles" refer to empty slots where no useful work is performed and misprediction of a branch happens when the CPU incorrectly guesses the outcome of a conditional branch (e.g the result of an if statement), mispredictions lead to bubbles as the pipeline is cleared and refilled

## Sort Order in Column Storage
In a column store, it doesn't necessarily matter in which order the rows are stored

It's easiest to store them in the order in which they were inserted, since then inserting a new row just means appending to each of the column files

However, we can choose to impose an order, like we did with SSTables previously, and use that as an indexing mechanism

Note that it wouldn't make sense to sort each column independently, because then we would no longer know which items in the columns belong to the same row, we can only reconstruct a row because we know that the *k*th item in one column belongs to the same row as the *k*th item in another column

Rather, the data needs to be sorted an entire row at a time, even though it is stored by column, the administrator of the database can choose the columns by the  which the table should be sorted, using their knowledge of common queries

For example, if queries often target date ranges, such as the last month, it might make sense to make `date_key` the first sort key, then the query optimizer can scan only the rows from the last month, which will be much faster than scanning all rows

A second column can determine the sort order of any rows that have the same value in the first column, for example if `date_key` is the first sort key it makes sense for `product_sk` to be the second sort key so that all sales for the same product on the same day are grouped together in storage, that will help queries that need to group or filter sales by product within a certain date range

Another advantage of sorted order is that it can help with compression of columns, if the primary sort column does not have many distinct values, then after sorting, it will have long sequences where the same value is repeated many times in a row, a simple run-length encoding, like we used for the bitmaps could compress that column down to a few kb even if the table has billions of rows

That compression effect is strongest on the first sort key, the second and third sort keys will be more jumbled up, and thus not have such long runs of repeated values

Columns further down the sorting priority appear in essential random order, so they probably won't compress as well, but having the first few columns sorted is still a win overall

### Several Different Sort Orders
A clever extension of this idea was introduced in C-Store and adopted in the commercial data warehouse Vertica

Different queries benefit from different sort orders, so why not store the same data sorted in several different ways?

Data needs to be replicated to multiple machines anyway so you don't lose that data if one machine fails, you might as well store that redundant data sorted in different ways so that when you're processing a query, you can use the version that best fits the query pattern

Having multiple sort orders in a column-oriented store is a bit similar to having multiple secondary indexes in a row-oriented store

But the big difference is that the row-oriented store keeps every row in one place, and secondary indexes just contain pointers to the matching rows

In a column store, there normally aren't any pointers to data elsewhere, only columns containing values

## Writing to Column-Oriented Storage
These optimizations make sense in data warehouses, because most of the load consists of large read-only queries run by analysts

Column-oriented storage, compression, and sorting all help to make those read queries faster, however they have the downside of making writes more difficult

An update-in-place approach, like B-trees use, is not possible with compressed columns, if you want to insert a row in the middle of a sorted table you would most likely have to rewrite all the column files, as rows are identified by their position within a column, the insertion has to update all columns consistently

Fortunately, LSM-trees are a good solution, all writes first go to an in-memory store, where they are added to a sorted structure and prepared for writing to a disk

It doesn't matter whether the in-memory store is row-oriented or column-oriented when enough writes have accumulated, they are merged with the column files on disk and written to new files in bulk, this is essentially what Vertica does

Queries need to examine both the column data on disk and the recent wires in memory, and combine the two

However, the query optimizer hides this distinction from the user, from an analysts point of view, data that has been modified with inserts, updates, or deletes, is immediately reflected in subsequent queries

## Aggregation: Data Cubes and Materialized Views
Another aspect of data warehouses that is worth mentioning is *material aggregates*

As discusses earlier, data warehouse queries often involve an aggregate function such as `COUNT`, `SUM`, `AVG`, `MIN`, or `MAX` in SQL

If the same aggregates are used by many different queries, it can be wasteful to crunch through the raw data every time, why not cache some of the counts or sums that queries use most often?

One way of creating such a cache is a *materialized view*, in a relational data model it is often defined like a standard view: a table-like object whose contents are the results of some query

The difference is that a materialized view is an actual copy of the query results, written to disk, whereas a virtual view is just a shortcut for writing queries

When you read from a virtual view, the SQL engine expands it into the view's underlying query on the fly and then processes the expanded query

When the underlying data changes, a materialized view needs to be updated, because it is a denormalized copy of data, the database can do that automatically, but such updates make writes more expensive, which is why material views are not often used in OLTP databases

In read-heavy data warehouses they can make more sense

*Note:* Instead of writing a complex query over and over, you can define it once as a view and simply query the view as if it were a table, this is a *virtual view*, it is not so correlated to a *materialized view* that actually caches results onto the disk

A common special case of a materialized view is known as a *data cube* or OLAP *cube*, it is a grid of aggregates grouped by different dimensions shown below

We can use *data cubes* like a **prefix sum array** to quickly find values!

![Image](<photos/aggregate_cubes.png>)

# Summary
In this chapter we tried to get to the bottom of how databases handle storage and retrieval

What happens when you store data in a database, and what does the database do when you query for the data again later?

On a high level, we saw that storage engines fall two broad categories: those optimized for transaction processing (OLTP), and those optimized for analytics (OLAP), there are big differences between the access patterns in those use case
- OLTP systems are typically user-facing, which means that they may see a huge volume of requests, in order to handle the load, applications usually only touch a small number of records in each query, the application requests records using some kind of key, and the storage engine uses an index to find the data for the requested key, disk seek time is often the bottle neck here
- Data warehouses and similar analytic systems are less well known, because they are primarily used by business analysts, not by end users, they handle a much lower volume of queries than OLTP systems, but each query is typically very demanding requiring millions of records to be scanned in a short time, disk bandwidth is often the bottleneck here, and column-oriented storage is an increasingly popular solution for this kind of workload

*Note*: Disk bandwidth refers to the maximum rate at which data can be read from or written to a storage device (like an HDD, SSD, or NVMe drive) **AFTER** the head has positions itself, disk seek time in the delay incurred whole the disk's read/write head moves to the correct track

On the OLTP side, we saw storage engines from two main schools of thought
- The log-structured school which only permits appending to files and deleting obsolete files, but never updates a file that has been written, bitcask, SSTables, LSM-trees, LevelDB, Cassandra, HBase, Lucene, and others belong here
- The update-in-place school which treats the disk as a set of fixed-size pages that can be overwritten, B-trees are the biggest example of this and is used in all major relational databases and many non-relational ones

Log-structured storage engines are a comparatively recent development. Their key idea is that they systematically turn random-access writes into sequential writes on disk, which enables higher write throughput due to the performance characteristics of hard drives and SSDs

We then took a detour from the internals of storage engines to look at the high-level architecture of a typical data warehouse. This background illustrated why analytic workloads are so different from OLTP: when your queries require sequentially scanning across a large number of rows, indexes are much less relevant, instead it becomes important to encode data very compactly, to minimize the amount of data that the query needs to read from disk, and we discussed how column-oriented storage helps achieve this goal