# Formats for Encoding and Decoding
Programs usually work with data in (at least) two representations: 
1. In memory, data is kept in objects, structs, lists, arrays, hash tables, trees, and so on, these data structures are optimizes for efficient access and manipulation by the CPU (typically using pointers)
2. When you want to write data to a file or send it over the network, you have to encode it as some kind of self-contained sequence of bytes(for example, a JSON document), since a pointer wouldn't make sense to any other process, this sequence-of-bytes representation looks quite different from the data structures normally used in memory

Thus we need some kind of translation between the two representations, this translation from in-memory representation to a byte sequence is called *encoding* (also known as *serialization* or *marshalling*), and the reverse is called *decoding* (*parsing*, *deserialization*, or *unmarshalling*)

*Note*: *Serialization* is unfortunately also used in the context of transactions, with a completely different meaning, to avoid overloading the word we will stick with *encoding*, even though serialization is perhaps a more common term

As this is such a common, there are a myriad of different libraries and encoding problems to choose from