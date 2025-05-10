# Introduction
Applications inevitably change over time, features are added or modified as new products launch, user requirements become better understood, or business circumstances change

In chapter 1, we introduced the idea of *evolvability*, we should aim to build systems that make it easy to adapt to change

In most cases, a change to an application's feature also requires a change to data that it stores, perhaps a new field or a record type needs to be captured or perhaps existing data needs to be presented in a new way

The data models we discussed have different ways of coping with such change, relational databases generally assume that all data in the database conforms to one schema: although that schema can be changed, there is exactly one schema in force at any point in time

By contrast, schemaless databases don't enforce a schema, so the database can contain a mixture of older and newer data formats written at different times

When a data format or schema changes, a corresponding change to application code often needs to happen, however ina  large application code changes cannot happen instantaneously

With server-side applications you may want to perform a *rolling upgrade* deploying the new version to a few nodes at a time, this allows new versions to be deployed without service downtime

With client-side applications you're at the mercy of the user who may not install the update for some time, this means that old and new versions of the code and old and new data formats may potentially all co-exist at the same time

In order for the system to continue running smoothly, we need to maintain compatibility in both directions:
- *Backward compatibility*: Newer code can read data written by older code
- *Format compatibility*: Older code can read data written by newer code

Backward compatibility is normally not hard to achieve, as author of the newer code you know the format of data written by older code, but forward compatibility can get tricky

We will look at several formats for encoding data, this includes JSON, XML, Protocol Buffers, Thrift, and Avro

In particular, we will look at how they handle schema changes and support old and new data, we will discuss how those formats are used for data storage and communication: in web services, Representational State Transfer (REST), remote procedure calls (RPC), and message-passing systems such as actors and message queue

# Formats for Encoding Data 
Programs usually work with data in (at least) two different representations:
1. In memory, data is kept in objects, structs, lists, arrays, hash tables, trees, and so on
2. When you want to write data to a file or send it over the network you have to encode it as some kind of self-contained sequence of bytes (for example, a JSON document), since a pointer wouldn't make sense to any other process, this sequence-of-bytes looks different from data structures normally used in memory

Thus, we need some kind of translation between the two representations, the translation from in-memory  representation to a byte sequence is called *encoding* (also known as *serialization* or *marshalling*), and the reverse is called *decoding* (*parsing*, *deserialization*, or *unmarshalling*)

## Language-Specific Formats
Many programming languages come with built-in support for encoding in-memory objects into byte sequences

For example, Java has `java.io.Serializable`, Ruby has `Marshal`, and Python has `pickle`, these encoding libraries are very convenient because they allow in-memory objects to be saved and restored with minimal additional coding, however they also have a number of problems:
- The encoding is often tied to a particular programming language and reading the data in another language is difficult, if you store or transmit data in such an encoding, you are committing yourself to your current programming language for potentially a very long time and precluding integration your systems with those of other organizations
- In order to restore data in the same object types, the decoding process needs to be able to instantiate arbitrary classes, this is frequently a source of security problems: if an attacker can get your application to decode an arbitrary byte sequence, they can instantiate arbitrary classes which allows them to remotely execute code
- Versioning and efficiency are afterthoughts

For these reasons, it's generally a bad idea to use your language's built-in encoding for anything other than very transient purposes

## JSON, XML, and Binary Variants
Moving to standardized encodings that can be written and read by many programming languages, JSON and XML are obvious contenders

They are widely known, widely supported, and almost widely disliked, XML is often criticized for being too verbose and unnecessarily complicated

JSON's popularity is mainly due to its built-in support for web browsers (by virtue of being a subset of Javascript) and simplicity rather than XML, CSV is another popular language-independent format, albeit less powerful

JSON, XML, and CSV are textual formats and thus somewhat human-readable, besides the superficial syntactic issues they also have some subtle problems
- There is a lot of ambiguity around the encoding of numbers, in XML and CSV you cannot distinguish between a number and a string that happens to consist of digits, JSON distinguishes strings and numbers but it doesn't distinguish integers and floating point numbers, and it doesn't specify precision
- JSON and XML have good support for Unicode character strings, but they don't support binary strings (a sequence of bytes without a character encoding), binary strings are a useful feature so people get around this limitation by encoding the binary data as text using Base64, the schema then used to indicate that the value should be interpreted as Base64-encoded, this works but increases the data size by 33%
- There is optional schema support for powerful XML and JSON, these schema languages are quite powerful and thus quite complicated to learn and implement
- CSV does not have any schema so it is up the application to define the meaning of each row and column, if an application change adds a new row or column you have to handle that change manually

Despite these flaws, JSON, XML, and CSV are good enough for many purposes, it's likely that they will remain popular, especially as data interchange formats, in these situations as long as people agree on what the format is, it doesn't matter how pretty or efficient the format is, the difficulty is getting different organization to agree on *anything*

### Binary Encoding
For data that is used only internally within your organization, there is less pressure to use a lowest-common-denominator encoding format

For example, you could choose a format that is more compact or faster to parse, for a small dataset the grains are negligible, but once you get into the terabytes, the choice of data format can have a big impact

JSON is less verbose than XML but both still use a lot of space compared to binary formats, hence the development of binary encodings for JSON such as MessagePack, BSON, BJSON, UBJSON, BISON, and Smile and for XML, WBXML and Fast Infoset

Say we look at MessagePack, a binary encoding for JSON, we use the example below
```json
{
    "userName": "Martin",
    "favoriteNumber": 1337,
    "interests": ["daydreaming", "hacking"]
}
```

1. The first byte, `0x83` indicates that what follows is an object with three fields, they are split into two nibbles (4-bit halves), high nibble = 0x8 and low nibble = 0x3, the high nibble is the *type code* for example 1000 xxxx means "fixmap" a small map/object whose size fits in 4 bits, and the low nibble is the *length* or *count*, here it means this fixmap has 3 key-value pairs
2. The second byte, `0xa8` indicates that whatever follows is a string that is 8 bytes long
3. The next eight bytes are the field name `userName` in ASCII, since the length was indicated previously there's no need for any marker to tell us where the string ends
4. The next seven bytes encode the six-letter string value `Martin` with a prefix `0xa6` and so on

The binary encoding is 66 bytes long, which is only a little less tha the 81 bytes taken by the textual JSON encoding (whitespace removed), all the binary encodings of JSON are similar in this regard, it's not clear whether such a small space reduction is worth the loss of human-readability

In the following sections we will encode the same record in just 32 bytes!

Below is the JSON fully encoded in MessagePack
![MessagePack Encoding](<photos/messagepack_encoding.png>)