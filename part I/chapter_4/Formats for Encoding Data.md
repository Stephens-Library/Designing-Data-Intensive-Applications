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

## JSON, XML, and Binary Variants
Moving to standardized encoding that can be written and ready by many programming languages, JSON and XML are the obvious contenders

Tey are widely known, widely supported, and almost as widely disliked

XML is often criticized for being too verbose and unnecessarily complicated and JSON's popularity is mainly due to its built-in support in web browsers and simplicity relative to XML

CSV is another popular language-independent format, though less powerful

JSON, XML, and CSV are textual formats, and thus somewhat human-readable (although the syntax is a popular topic of debate), besides the superficial syntactic issues, they also have some subtle problems
- There is a lot of ambiguity around the encoding of numbers, in XML and CSV you cannot distinguish between a number and a string that consists of digits (except by referring to an external schema), JSON distinguishes strings and numbers, but doesn't distinguish integers and floating-point numbers, and doesn't specify a precision

This is a problem when dealing with large numbers, for example integers greater than 2^53 cannot be exactly represented in an IEEE 754 double-precision floating-point number, so such numbers become inaccurate when parsed in a language that uses floating-point numbers (such as JavaScript)

An example of numbers larger than 2^53 occurs on Twitter which uses a 64-bit number to identify each tweet, the JSON returned by Twitter's API includes tweet IDs twice, once as a JSON number and once as a decimal string, to work around the fact that the numbers are not correctly passed by JavaScript applications

- JSON and XML have good support for Unicode characters, but they don't support binary strings (sequences of bytes without a character encoding), binary strings are a useful feature, so people get around this limitation by encoding the binary data as text using Base64

The schema is then used to indicate that the value should be interpreted as Base64-encoded, this works, but it's somewhat hacky and increases the data size by 33%

- There is optional schema support for both XML and JSON, these schema languages are quite powerful, and thus quite complicated to learn and implement

Use of XML schemas is fairly widespread, but many JSON-based tools don't bother using schemas, since the correct interpretation of data depends on information in the schema, applications that don't use XML/JSON schema need to potentially hardcode the appropriate encoding/decoding logic instead

- CSV does not have any schema, so it is up to the application to define the meaning of each row and column, if an application change adds a new row or column, you have to handle that change manually

CSV is also quite a vague format (what happens if a value contains a comma or a newline character), although its escaping rules have been formally specified, not all parsers implement them correctly

Despite these flaws, JSON, XML, and CSV are good enough for many purposes

It is likely they will remain popular, especially as data interchange formats

In these situations, as long as people agree on what the format is, it often doesn't matter how pretty or efficient the format is, the difficulty of getting different organizations to agree on *anything* outweighs most other concerns