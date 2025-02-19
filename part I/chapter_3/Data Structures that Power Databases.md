

## Data Structures that Power Your Database
Consider the world's simples database, implemented by two Bash functions:
```bash
db_set() {
    echo "$1,$2" >> database
}

db_get() {
    grep "^$1," database | sed -e "s/^$1,//" | tail -n 1
}
```

These two functions implement a key-value store, you can call `db_set` key value, which will store `key` and `value` in the database

The key and value can be (almost) anything you like, for example the value could be a JSON document

You can then call `db_get` key, which looks up the most recent value associated with that particular key and returns it

The underlying storage format is very simple: a text file where each line contain a key-value pair, separated by a comma (roughly like a CSV file, ignoring escaping issues)

Every call to `db_set` appends to the end of the file, so if you update a key several times, the old versions of the value are not overwritten, you need to look at the last occurrence of a key in a file to find the latest value

Our `db_set` function actually has a pretty good performance for something that is so simple, because appending to a file is generally very efficient, similarly to what `db_set` does, many databases internally use a `log` which is an append-only data file

Real databases have more issues to deal with (such as concurrency control, reclaiming disk space, and handling errors) but the basic principle is the same

Logs are incredibly useful, the word *log* is often used to refer to application logs, where an application outputs text that describes that's happening

In this book, *log* is used in a more general sense, an append-only sequence of records, it doesn't have to be human-readable; it might be binary and intended only for programs to read

On the other hand, our `db_get` function has terrible performance if you have a large number of records, every time you want to look up a key, `db_get` has to scan the entire database file

In algorithmic terms, the cost of lookup is `O(n)` if you double the number of records `n`, the lookup takes twice as long

In order to efficiently find the value for a particular key in the database, we need a different data structure, an *index*

In this chapter, we will look t a range of indexing structures and see how they compare; the general idea behind them is to keep some additional metadata on the side, which acts as a signpost and helps you locate the data you want

An index is an *additional* structure that is derived from the primary data, many databases allow you to add and remove indexes, and this doesn't affect the contents of the database; it only affects the performance of queries

Maintaining additional structures incurs overhead, especially on writes, for writes it's hard to beat the performance of simply appending to a file, because that's the simplest write operation, any kind of index usually slows down writes, because the index also needs to be updated when the data is written

This is an important trade-off in storage systems: well-chosen indexes speed up read queries, but every index slows down writes

For this reason, databases don't usually index everything by default, but require you to choose indexes manually, using your knowledge of the application's typical query patterns

You can then choose the indexes that give your application the greatest benefit, without introducing more overhead than necessary
