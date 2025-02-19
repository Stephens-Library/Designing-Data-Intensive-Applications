# Storage and Retrieval
On the most fundamental level, a database needs to do two things when you give it some data ,it should store the data, and when you ask it again later, it should give the data back to you

In chapter 2 we discussed data models and query languages, the format in which you give the database your data, and the mechanism by which you can ask for it later, in this chapter we discuss the same from the database's point of view: how we can store the data that we're given, and how we can find it again when we're asked for it

We will examine two families of storage engines: *log-structured* storage engines, and *page-oriented* storage engines such as B-trees