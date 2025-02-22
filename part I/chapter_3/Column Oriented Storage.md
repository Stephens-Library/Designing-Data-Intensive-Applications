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