# Storage and Retrieval
On the most fundamental level, a database needs to do two things when you give it some data ,it should store the data, and when you ask it again later, it should give the data back to you

In chapter 2 we discussed data models and query languages, the format in which you give the database your data, and the mechanism by which you can ask for it later, in this chapter we discuss the same from the database's point of view: how we can store the data that we're given, and how we can find it again when we're asked for it

We will examine two families of storage engines: *log-structured* storage engines, and *page-oriented* storage engines such as B-trees

## SSTable and LSM-Trees
In the figure above, each log-structured storage segment is a sequence of key-value pairs

These pairs appear in the order that they were written, and values later in the log take precedence over values for the same key earlier in the log, apart from that, the order of key-value pairs in the file does not matter

Now we can make a simple change to the format of our segment files: we require that the sequence of key-value pairs is *sorted by key*

We call this format *Sorted String Table* or *SSTable* for short, we also require that each key only appears once within each merged segment file, SSTables have several big advantages over log segments with hash indexes:

1. Merging segments is simple and efficient, even if the files are bigger than the available memory, the approach is like the one used in the *merge sort* algorithm

You start reading the input files side by side, look at the first key in each file, copy the lowest key to the output file, and repeat, this produces a new merge segment file, also sorted by key

Furthermore, the reason is uses minimal memory is because you only keep a small part of each file in memory (the current block), the total memory used is proportional to the number of files, not the total file size, that is stored on disk, but we can still access it sequentially not randomly as we have pointers to each file

![Image](<photos/SSTable.png>)
*This figure shows merging several SSTable segments, retaining only the most recent value for each key*

What if the same key appears in several input segments? Remember that each segment contains all the values written to the database during some period of time

This means that all the values in one input segment must be more recent than all the values in the other segment (assuming we always merge adjacent segments), when multiple segments contain the same key, we can keep the value from the most recent segment and discard the values in older segments

2. In order to find a particular key in the file, you no longer need to keep an index of all the keys in memory, say you don't know the exact offset of the key in the segment file, but you do now the offsets for two keys in between the target key, then you can jump to the offset of the first key and scan from here

You still need an in-memory index to tell you the offsets for some of the keys, but it can be sparse: one key for every few kilobytes of segment file is sufficient, because a few kilobytes can be scanned very quickly!

3. Since read requests need to scan over several key-value pairs in the requested range anyway, it is possible to group those records into a block and compress it before writing it to disk, each entry of the sparse in-memory index then points at the start of a compressed block, besides saving disk space, compression also reduces the I/O bandwidth use

### Constructing and Maintaining SSTables
How do we ge3t data to be sorted by key in the first place? Our incoming writes can occur in any order

Maintaining a sorted structure on disk is possible (B-Trees), but maintaining it in memory is much easier

There are plenty of well-known tree data structures you can sue, such as red-black trees, or AVL trees

With these data structure, you can insert keys in any order and read them back in sorted order, we can now make our storage engine work as follows:
- When a write comes in, add it to an in-memory balanced tree structure (e.g red-black tree), this in-memory tree is sometimes called a *memtable*
- When the memtable gets bigger than some threshold, typically a few megabytes, write it out to a disk as an SSTable file, this ca be done efficiently because three already maintains the key-value pairs sorted by key, the new SSTable file becomes the most recent segment of the database
- In order to serve a read request, first try to find the key in the memtable, then in the most recent on-disk segment, then in the next-older segment, etc
- From time to time, run a merging and compaction process in the background to combine segment files and to discard overwritten and deleted values

### Making an LSM-tree out of SSTables
The algorithm described here is essentially what is used in LevelDB and RocksDb, key-value storage engine libraries are designed to be embedded into other applications

Among other things, LevelDB can be used in Riak as an alternative to Bitcast, similar storage engines are used in Cassandra and HBase

Originally, this indexing structure was described b yPatrick O'Neil et al. under the name *Log-Structured Merge-Tree* (or LSM-Tree), building on earlier work on log-structured filesystems

Storage engines that are based on this principle of merging and compacting sorted files are often called LSM storage engines

Lucene, an indexing engine for full-text search used by Elastisearch and Solr, uses a similar method for storing its *term dictionary*

A full-text index is much more complex than a key-value index but is based on a similar idea: given a word in a search query, find all the documents (web pages, product descriptions, etc) that mention the word

This is implemented with a key-value structure where the key is a word (a *term*) and the value is the list of IDs of all the documents that contain the word (the *postings list*)

In Lucene, this mapping from term to postings list is kept in SSTable-like sorted files, which are merged in the background as needed

*My Explanation*: Essentially, the first important thing to note that an LSM tree is **NOT** a tree data structure the way a binary search tree is, a LSM tree refers to **the hierarchical organization of data across multiple levels**

Here is how the levels/hierarchy is broken down
1. Memtable: In-memory structure implemented with a balanced tree that keeps the keys sorted
2. SSTable: When the memtable reaches a certain size, it is frozen and flushed to disk as an SSTable **every time a flush occurs it writes to a new SSTable file, overtime you'll accumulate multiple SSTable files, and the background compaction process will merge them**
3. On-disk: The SSTables are organized into multiple levels via background compaction where SSTables from a lower level are merged into a higher level

*A deeper look*: I personally thought why don't they decide to store the files into one big file, wouldn't this reduce the searching radius?

While it does, it is a tradeoff, you may have to perform a few more binary searches, but using multiple immutable files avoids the complexity and performance penalities of constantly updating one giant file and simplifies crash recovery

### Performance Optimizations
As always, a lot of detail goes into making a storage engine perform well in practice

For example, the LSM-tree algorithm can be slow when looking up keys that do not exist in the database, you have to check the memtable, then the segments all the way back to the oldest before you can be sure that the key does not exist

In order to optimize this kind of access, storage engines often use additional *Bloom filters*

A bloom filter is a memory-efficient data structure for approximating the contents of a set, it can tell you if a key does not appear in the database, and thus saves many unnecessary disk reads for non-existent keys

There are also different strategies to determine the order and timing of how SSTables are compacted and merged

The most common options are *size-tiered* and *leveled* compaction, LevelDB and RocksDB use leveled compaction, HBase uses size-tiered and Cassandra supports both

In size-tiered compaction, newer and smaller SSTables are successively merged into older and larger SSTables

In leveled compaction, the key range is split up into smaller SSTables and older data is moved into separate "levels," which allows the compaction to proceed more incrementally and use less disk space

Even though there are many subtleties, the basic idea of LSM-trees (keeping a cascade of SSTables that are merged in the background) is simple and effective

Even when the dataset is much bigger than the available memory it continues to work well, since data is stored in sorted order, you can efficiently perform range queries, and because the disk writes are sequential the LSM-tree can support remarkably high write throughput