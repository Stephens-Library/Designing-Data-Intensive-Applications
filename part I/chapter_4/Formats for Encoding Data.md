# Formats for Encoding and Decoding
Programs usually work with data in (at least) two representations: 
1. In memory, data is kept in objects, structs, lists, arrays, hash tables, trees, and so on, these data structures are optimizes for efficient access and manipulation by the CPU (typically using pointers)
2. When you want to write data to a file or send it over the network, you have to encode it as some kind of self-contained sequence of bytes(for example, a JSON document), **since a pointer wouldn't make sense to any other process as it doesn't understand your program's memory addresses**, this sequence-of-bytes representation looks quite different from the data structures normally used in memory

Thus we need some kind of translation between the two representations, this translation from in-memory representation to a byte sequence is called *encoding* (also known as *serialization* or *marshalling*), and the reverse is called *decoding* (*parsing*, *deserialization*, or *unmarshalling*)

*Note*: *Serialization* is unfortunately also used in the context of transactions, with a completely different meaning, to avoid overloading the word we will stick with *encoding*, even though serialization is perhaps a more common term

As this is such a common, there are a myriad of different libraries and encoding problems to choose from

Long story short, **encoding creates a common format everyone can understand (interoperability)**

## Language-Specific Formats
Many programming languages come with built-in support for encoding in-memory objects into byte sequences

FOr example, Java has `java.io.Serializable`, Ruby has `Marshal`, Python has `Pickle`, and so on

Many third-party libraries also exist such as `Kryo` for Java

These encoding libraries are very convenient, because they allow in-memory objects to be saved and restored with minimal additional code, however they also have a number of deep problems:
- The encoding is often tied to a particular programming language, and reading the data in another language is difficult
- In order to restore data in the same object types, the decoding process needs to be able to instantiate arbitrary classes, this is frequently a source of security problems: if an attacker can get your application code to decode an arbitrary byte sequence, they can instantiate arbitrary classes, which in turn often allows them to do terrible things such as remotely executing arbitrary code
- Versioning data is often an afterthought in these libraries: as they are intended for quick and easy encoding of data, they often neglect the inconvenient problems of forward and backward compatibility
- Efficiency (CPU time taken to encode or decode, and the size of the encoded structure) is also often an afterthought

For these reasons it's generally a bad idea to use your language's built-in encoding for anything other than very transient purposes