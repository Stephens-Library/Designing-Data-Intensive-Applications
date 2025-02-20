

# Data Structures that Power Your Database
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

## Hash Indexes
Let's start with indexes for key-value data, this is not the only kind of data you can index, but it's very common, and it's a useful building block for more complex indexes

Key-value stores are quite similar to the *dictionary* type that you can find in most programming languages, and which is usually implemented as a hash map

### How Hashmaps Work
Under the hood, a hash map computes a hash function on a key to generate an index, this index points to where the corresponding value is stored in memory!

However, in-memory structures like hash maps are volatile and limited in size, so why not write data on disk with an append-only file?

Let's say our data storage consists only of appending to a file, as in the preceding example, then the simplest possible indexing strategy is to keep an in-memory hash map where every key is mapped to a **byte offset** in the data file

Whenever you append a new key-value pair to the file, you also update the hash map to reflect the offset of the data you just wrote, when you want to look up a value, use the hash map to find the offset in the data file, seek to that location, and read the value

![Image](<photos/byte_offset.png>)
*This figure shows a log of key-value pairs in a CSV-like format, indexes with an in-memory hash map*

As described so far, we only ever append to a file, so how do we avoid running out of disk space?

A good solution is to break the log into segments of a certain size by closing a segment file when it reaches a certain size, and making subsequent writes to a new segment file, we can then perform *compact* on these segments

Compaction means throwing away duplicate keys in the log and keeping only the most recent update for each key

![Image](<photos/compaction.png>)
*This figure shows the process of compaction of a key-value update log, retaining only the most recent value for each key*

Moreover, since compaction often makes segments much smaller, we can also merge several segments together as the same time as performing the compaction, after the merging process is complete, we switch read requests to using the new merged segment instead of the old segment, and then the old segment files can simply be deleted

Of course, this will mean that the byte offsets for the key-value pairs will likely change, and you'll need to update your in-memory index such that each key points to the correct new offset in the merged segment

![Image](<photos/multiple_compaction.png>)
*This figure shows compaction and segment merging simultaneously*

Each segment now has its own in-memory hash table, mapping keys to file offsets

In order to find the value for a key, we first check the most recent segment's hash map, if the key is not present, we check the second-most recent segment, and so on

The merging process keeps the number of segments small, so lookups don't need to check many hash maps

Here is a brief overview of some issues that are important in real implementation
- *File format*: CSV is not the best format for a long, it's faster and simpler to use a binary format that first encodes the length of a string in bytes, followed by the raw string
- *Deleting records*: If you want to delete a key and its associated value, you have to append a special deletion record to the data file (sometimes called a *tombstone*), when log segments are merged, the tombstone tells the merging process to discard any previous values for the deleted key
- *Crash recovery*: If the database is restarted, the in-memory hash map are lost, in theory you can recover the segment's hash,ap by reading the entire file segment from beginning to end and noting the offset of the most recent value for every key as you go along, though you can speed up the recovery process by storing a snapshot of each segment's hash map on disk, which can be loaded into memory more quickly
- *Partially written records*: The database may crash at any time, including halfway through appending a record to a log
*Concurrency control*: As writes are appended to the log in a strictly sequential order, a common implementation choice is to have only one writer thread, data file segments are append-only and otherwise immutable, so they can be read concurrently by multiple threads

At first, an append-only log seems wasteful at first, why don't you update the file in place and overwrite the old value with the new value? Below are some reasons an append-only design turns out to be good
- Appending and segment merging are sequential write operations, which are generally much faster than random writes, especially on magnetic spinning-disk hard drivers, to some extent sequential writes are also preferable on flash-based solid state drives
- Concurrency and crash recovery are much simpler if segment files are append-only or immutable, for example, you don't have to worry about the case where a crash happened while a value was being overwritten, leaving you with a file containing part of the old and new value spliced together
- Merging old segments avoids the problem of data files getting fragmented over time

However the has table index also has limitations
- The hash table must fit in memory, so if you have a very large number of keys, you're out of luck, in principle you could maintain a hash map on a disk, but it is difficult to make an on-disk hash map perform well, it requires a lot of random access I/O, it is expensive to grow when it becomes full, and hash collisions require fiddly logic
- Range queries are not efficient, for example you cannot easily scan over all keys between `kitty00000` and `kitty99999` you'd have to look up each key individually in the hash maps

*Note: On-disk operations are much slower due to random access times because the storage medium (mechanical hard drives) isn't designed to jump around quickly, on a spinning disk, the read/write head must physically move to the correct location on the disk, when accessing data randomly, the head must move to many different positions, which is slower than reading data sequentially in one block*

In the next section, we will look at an indexing structure that doesn't have those limitations