# Data Models and Query Languages
Data models are perhaps the most important part of developing software, because they have such a profound effect: not only on how the software is written, but also on how we *think about the problem* we are solving.

Most applications are built by layering one data model on top of another. For each layer, the key question is: how is it *represented* in terms of the next-lowest layer? For example:
1. As an application developer, you look at the real world and model it in terms of objects, data structures, and APIs that manipulate those data structures, those structures are often specific to your application
2. When you want to store those data structures, you express them in terms of a general-purpose data model, such as JSON or XML documents, tables in a relational database or graph model
3. The engineers who built your database software decided on a way of presenting that JSON/XML/relational/graph data in terms of bytes in memory, on disk, or on a network. The representation may allow the data to be queried, searched, manipulated, and processed in various ways
4. On yet lower levels, hardware engineers have figured out how to represent bytes in terms of electrical currents, pulses of light, magnetic fields, and more

In a complex application there may be more intermediary levels, such as APIs built upon APIs, but the basic idea is still the same: each layer hides the complexity of the layers below it by providing a clean data model, these abstractions allow different groups of people to work together effectively

There are different kinds of data models, and every data model embodies assumptions about how it is going to be used, some kinds of usage are easy and some are not supported

Some operations are fast and some perform badly; some data transformations feel natural and some are awkward, it can take a lot of effort to master just one data model

Building software is hard enough, even when working with just one data model and without worrying about its inner workings, but since the data model has such a profound effect on what the software above it can and can't do, it's important to choose one that is appropriate to the application

We will look at a range of general-purpose data models for data storage and querying, in particular we will compare the relational data model, the document model, and a few graph based data models, in the next chapter we will discuss how storage engines work; that is, how these data models are actually implemented
