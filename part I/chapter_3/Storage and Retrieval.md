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