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