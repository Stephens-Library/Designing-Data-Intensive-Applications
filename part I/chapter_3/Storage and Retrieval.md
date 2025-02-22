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

## B-Trees
The log-structured indexes we have discussed so far are gaining acceptance, but they are not the most common type of index

The most widely used indexing structure is quite different: The B-tree

Like SSTables, B-trees keep key-value pairs sorted by key, which allows efficient key-value lookups and range queries, but that's where the similarity ends: B-trees have a very different design philosophy

The log-structured indexes we saw earlier break the database down into variable-size segments, typically several megabytes or more in size, and always write a segment sequentially

By contrast, B-trees break the data down into fixed-size *blocks* or *pages*, traditionally 4 KB in size, and read or write one page at a time

This design corresponds more closely to the underlying hardware, as disks are also arranged in fixed-size blocks

Each page can be identified using an address or location, which allows one page to refer to another, similar to a pointer, but on disk instead of in memory, we can use these page references to construct a tree of pages

![Image](<photos/tree_of_pages.png>)
*This figure depicts looking up a key using a B-tree index*

One page is designated as the *root* of the B-tree; whenever you want to look up a key in the index, you start at the root

The page contains several keys and references to child pages, each child is responsible for a continuous range of keys and the keys between the references indicate where the boundaries between those ranges lie

In the photo above, we are looking for key 251, so we know that we need to follow the page reference between the boundaries 200 and 300, that takes us to similar-looking pages that further breaks down the 200-300 range into sub-ranges

Eventually we get down to a page containing individual keys (a *leaf page*), which either contains the value for each key inline or contains references to the pages where the values can be found

The number of references to a child page in one page of the B-tree is called the *branching factor*, in the photo above, the branching factor is six, in practice, the branching factor depends on the amount of space required to store the page references and the range boundaries, but typically is several hundred

If you want to update the value for an existing key in a B-tree, you search for the leaf page containing the key, change the value in that page, and write the page back to disk

If you want to add a new key, you need to find the page whose range encompasses the new key and add it to that page

If there isn't enough free space in the page to accommodate the new key, it is split into two half-full pages, and the parent page is updated to account for the new subdivision of key ranges

![Image](<photos/growing_b-tree.png>)

This algorithm ensures that the tree remains *balanced* a B-tree with *n* keys always has a depth of O(log n), most databases can fit into a Btree that is three or four levels deep, so you don't need to follow many page references to find the page you are looking for (A four-level tree of 4 KB pages with a branching factor of 500 can store up to
256 TB)

### Making B-Trees Reliable
The basic underlying write operation of a B-tree is to overwrite a page on disk with new data

It is assumed that the overwrite does not change the location of the page, i.e., all references to that page remains intact when the page is overwritten

This is in stark contrast to log-structured indexes such as LSM-trees, which only append to files but never modify files in place

You can think of overwriting a page on disk as an actual hardware operation, on a magnetic hard drive, this means moving the disk head to the right place, waiting for the right position on the spinning platter to come around, and then overwriting the appropriate section with new data

On SSDs, what happens is somewhat more complicated, due to the fact that an SSD must erase and rewrite fairly large blocks of a storage chip at a time

Some operations require several different pages to be overwritten, for example if you split a page because an insertion caused it to be overfull, you need to write the two pages that were split, and also overwrite their parent page to update the references to the two child pages

This is a dangerous operation as if the database crashes after only some pages have been overwritten, you might end up with a corrupted index (e.g an *orphan* page with no parent)

To make the database resilient to crashes, it's common for B-tree implementations to include an additional data structure on disk: a *write-ahead log* (WAL, also known as *redo log*)

This is an append-only file to which every B-tree modification must be written before it can be applied to the pages of the tree itself, when the database comes back up after a crash, this log is used to restore the B-tree back to a consistent state

An additional complication of updating pages in place is that careful concurrency control is required if multiple threads are going to access the B-tree at the same time, other a thread may see the tree in an inconsistent state

This is done by protecting the tree's data structure with *latches*, log structured approaches are simpler in this regard, because they do all the merging in the background

### B-Tree Optimizations
As B-trees have been around for so long, it's not surprising that many optimizations have been developed over the years, to mention just a few:
- Instead of overwriting pages and maintaining a WAL for crash recovery, some databases use a copy-on-write scheme, a modified page is written to a different location, and a new version of the parent pages in the tree is created, pointing at the new location

- In general, pages can be positioned anywhere on disk, there is nothing requiring pages with nearby key ranges to be nearby on disk, if a query needs to scan over  large part of the key range in sorted order, that page-by-page layout can be inefficient, because a disk seek may be required for every page that is read

Many B-tree implementations try to lay out the tree so that leaf pages appear in sequential order on disk, however it's difficult to maintain that order as the tree grows, by contrast since LSM-trees rewrite large segments of the storage in one go during merging, it's easier for them to keep sequential keys close to each other on disk

- Additional pointers have been added to the tree, for example, each leaf page may have references to its sibling pages to the left and right, which allows scanning keys in order without jumping back to parent pages

- B-tree variants such as *fractal trees* borrow from log-structured ideas to reduce disk seeks (and they have nothing to do with fractals)

*Note*: A disk seek refers to the time it takes for a hard drive's read/write head to move to the track on the disk where the desired data is stored

### Comparing B-Trees and LSM-Trees
Even though B-tree implementations are generally more mature than LSM-tree implementations, LSM-trees are also interesting due to their performance characteristics

As a rule of thumb, LSM-trees are typically faster for writes, whereas B-trees are thought to be faster for reads, reads are typically slower on LSM-trees because they have to check several different data structures and SSTables at different stages of compaction

However, benchmarks are often inconclusive and sensitive to details of the workload, you need to test systems with your particular workload in order to make a valid comparison

### Advantages of LSM-Trees
A B-tree index must write every piece of data at least twice: once to the write-ahead log, and once to the tree page itself

There is also overhead from having to write an entire page at a time, even if only a few bytes in that page changed up, some storage engines even overwrite the same page twice in order to avoid ending up with a partially updated page in the event of a power failure

*Note*: Disks are organized into fixed-size units (pages or blocks), even if only a few bytes change, you still have to rewrite the entire fixed-size because that's the smallest unit the disk can write

Log-structured indexes also rewrite data multiple times due to repeated compaction and merging of SSTables, this effect (where one write to the database causes multiple writes to the disk over the course of the database's lifetime) is called *write amplification*, it is of particular concern on SSDs which can only overwrite blocks a limited number of times before wearing out

In write-heavy applications, the performance bottleneck might be the rate at which the database can write to the disk, in this case, write amplification has a direct performance cost: the more that a storage engine writes to a disk, the fewer writes per second it can handle within the available disk bandwidth

Moreover, LSM-trees are typically able to sustain higher write throughput than B-trees, partly because they sometimes have lower write amplification (though this depends on the storage configuration and workload), and partly because they sequentially write compact SSTable files rather than having to overwrite several pages in the tree

This difference is particularly important on magnetic hard drives, where sequential writes are much faster than random writes

LSM-trees can be compressed better, and thus often produce smaller files on disk than B-trees, B-tree storage engines leave some disk space unused due to fragmentation: when a page is split or when a row cannot fit into an existing page, some space in a page remains unused, since LSM-trees are not page-oriented and periodically rewrite SSTables to remove fragmentation, they have lower storage overheads, especially when using leveled compaction

On many SSDs, the firmware internally uses a log-structured algorithm to turn random writes into , sequential writes on the underlying storage chips so the impact on the storage engine's write pattern is less pronounced


### Downsides of LSM-Trees
A downside of a log-structured storage is that the compaction process can sometimes interfere with the performance of ongoing reads and writes, even though storage engines try to perform compactions incrementally and without affecting concurrent access, disks have limited sources, so it can easily happen that a request needs to wait while the disk finishes an expensive compaction operation

The impact on throughput and average response time is usually small, but at higher percentiles, the response time of queries to log-structured storage engines can sometimes be quite high, and B-trees can be more predictable

Another issue with compaction arises at high write throughput: the disk's finite write bandwidth needs to be shared between the initial write and the compaction threads running in the background

When writing to an empty database, the full disk bandwidth can be used for the initial write, but the bigger the database gets, the more disk bandwidth is required for compaction

If write throughput is high and compaction is not configured carefully, the compaction may not be able to keep up with the rate of incoming writes, and the number of unmerged segments on disk would keep growing until you run out of disk space, and reads also slow down because they need to check more segment files

Typically, SSTable-based storage do not throttle the rate of incoming writes even if compaction cannot keep up so this needs to be explicitly monitored

An advantage of B-trees is that each key exists in exactly one place in the index, whereas a log-structured storage engine may have multiple copes of the same key in different segments, this aspect makes B-trees attractive in databases that want to offer strong transactional semantics: in many relational databases, transaction isolation is implemented using locks on ranges of keys and in a B-tree index, those locks can be directly attached to the tree

B-trees are very ingrained in the architecture of databases and provide consistently good performance for many workloads so it's unlikely that they will go away anytime, there is no quick and easy rule for determining which type of storage engine is better, so it's worth testing empirically

## Other Indexing Structures
So far, we have only discussed key-value indexes, which are like a *primary key* index in the relational model

A primary key uniquely identifies one row in a relational table, or one document in a document database, or one vertex in a graph database

Other records in the database can refer to that row/document/vertex by its primary key, and the index is used to resolve such references

It is also very common to have *secondary indexes*, in relational databases, you can create several secondary indexes on the same table using the `CREATE INDEX` command, and they are often crucial for performing joins efficiently

A secondary index can easily be constructed from a key-value index, the main difference is that keys are not unique, there might be many rows with the same key, this can be solved in two ways: either by making each value in the index a list of matching row identifiers or by making each key unique by appending a row identifier to it 

Either way, both B-trees and log-structured indexes can be used as secondary indexes

### Storing Values Within the Index
The key in an index is the thing that queries search for, but the value can be one of two things, it could be the actual row (document, vertex) in question, or it could be a reference to the row stored elsewhere

In the latter case, the place where rows are stored is known as a *heap file* and it stores data in no particular order

The heap file approach is common because it avoids duplicating data when multiple secondary indexes are present, each index just references a location in the heap file, and the actual data is kept in one place

When updating a value without changing the key, the heap file approach can be quite efficient: the record can be overwritten in place, provided that the new value is not larger than the old value

The situation is more complicated if the new value is larger, as it probably needs to be moved to a new location in the heap where there is enough space

In that case, either all indexes need to be updated to a point at the new heap location of the record, or a forwarding pointer is left behind in the old heap location

In some situations, the extra hope from the index to the heap file is too much of a performance penalty for reads, so it can be desirable to store the indexed row directly within an index, this is known as a *clustered index* for example in MySQL's InnoDB storage engine, the primary key of a table is always a clustered index, and secondary indexes refer to the primary key (rather than a heap file location)

A compromise between a clustered index (storing all row data within the index) and a non-clustered index (storing only references to the data within the index) is known as *covering index* or *index with included columns*, which stores *some* of a table's column

As with any kind of duplication of data, clustered and covering indexes can speed up reads, but they require additional storage and can add overhead on writes, databases also need to go to additional effort to enforce transactional guarantees, because applications should not see inconsistencies due to the duplication

Okay! Let me break that down

A **heap file** is where your table's full rows (all data) are stored on disk, when you look up data using an index, a typical index just gives you a pointers to where the row is in the heap file

A **non-clustered index**: It stores only the indexed columns plus a pointer to the full row in the heap file

A clustered index actually stores the **full row data** along with the key, this means the table's rows are physically stored in the order of the clustered index, it eliminates the extra step to hop from the index to the heap file, making read operations faster

**Covering index**: A non-clustered index that includes additional columns so that for certain queries, the index itself has all the data you need, eliminating extra disk access

*Note*: It is called a heap file because it stores records in an unsorted, unordered manner like a physical heap, **it is unrelated to the heap data structure**

### Multi-Column Indexes
The indexes discussed so far only map a single key to a value, it is not sufficient if we need to query multiple columns of a table simultaneously

The most common type of multi-column index is called a *concatenated index* which simply combines several fields into one key by appending one column to another

Multi-dimensional indexes are a more general way of querying several columns at once, which is particularly important for geo-spatial data

For example, a restaurant-search website may have a database containing the latitude and longitude of each restaurant, when a user is looking at the restaurants on a map, the website needs to search for all the restaurants within the rectangular map area that the user is currently viewing

This requires a two-dimensional range query like the following:
```sql
SELECT * FROM restaurants WHERE latitude > 51.4946 AND latitude < 51.5079 AND longitude > -0.1162 AND longitude < -0.1004;
```
A standard B-tree or LSM-tree index is not able to answer that kind of query efficiently, it can give you either all the restaurants in a range of latitudes or all the restaurants in a range of longitudes but not both simultaneously

One option is to translate a two-dimensional location into a single number using a space-filling curve, then to use a regular B-tree index, more commonly, specialized spatial indexes use R-trees using PostgreSQL's Generalized Search Tree indexing facility

### Full-Text Search and Fuzzy Indexes
All the indexes discussed so far assume that you have exact data and allow you to query for exact values of a key, or a range of values of a key with a sort order

What they don't allow you to do is a search for similar keys, such as misspelled words, such *fuzzy* querying requires different techniques

For example, full-text search engines commonly allow a search for one word to be expanded to include synonyms of the word, to ignore grammatical variations of words, and to search for occurrences of words near each other in the same document, and support various other features that depend on linguistic analysis of the text

To cope with typos in documents or queries, Lucene is able to search text for words within a certain edit distance

As mentioned in "Making an LSM-tree out of SSTables", Lucene uses a SSTable for its term dictionary, this structure requires a small in-memory index that tells queries at which offset in the sorted file they need to look for a key

In Lucene, the in-memory index is a finite state automation over the characters in the keys, similar to a *trie*, this automation can be transformed into a *Levenshtein automaton* which supports efficient search for words within a given edit distance

Other fuzzy search techniques go in the direction of document classification and machine learning