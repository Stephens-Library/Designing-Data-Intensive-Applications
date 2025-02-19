# Query Languages for Data
When the relational model was introduced, it included a new way of querying data: SQL is a *declarative* query language, whereas IMS and CODASYL queried the data using *imperative* code, so what exactly does this mean?

Many commonly used programming languages are imperative, for example if you have a list of animal species and wish to only return sharks you might write something like this
```javascript
function getSharks() {
    var sharks = [];
    for (var i = 0; i < animals.length; i++) {
        if (animals[i].family === "Sharks") {
            sharks.push(animals[i]);
        }
    }
    return sharks;
}
```
In the relational algebra, you would instead write: `sharks = σ_(family = "Sharks")(animals)` where the greek letter sigma (σ) is the selection operator returning only those animals that match the condition `family = "Sharks"`

When SQL was defined it followed the structure of the relational algebra fairly closely: 
```SQL 
SELECT * FROM animals WHERE family = 'Sharks';
```

An imperative language tells the computer to perform certain operations in a certain order, in a declarative language like SQL or relational algebra, you must specify the pattern of the data you want, what conditions the results must meet, and how you want the data to be transformed (sorted, grouped, aggregated), but not *how* to achieve that goal

It is up to the database system's query optimizer to decide which indexes and which join methods to use, and in which order to execute various parts of the query

A declarative language is more attractive because it is typically more concise and easier to work with than an imperative API, but more importantly, it also hides implementation details of the database engine, which makes it possible for the database system to introduce performance improvements without requiring any changes to queries, the fact that SQL is more limited in functionality gives the database much more room for automatic optimizations

Finally, declarative languages often lend themselves to parallel execution, today CPUs are getting faster by adding more cores, not by running at significantly higher clock speeds than before, imperative code is very hard to parallelize across multiple cores and machines because it specifies instructions that must be performed in a particular order

Declarative languages have a better chance of getting faster in parallel execution because they specify only the pattern of the results, not the algorithm that is used to determine the results

The database is free to use a parallel implementation of the query language if appropriate

## MapReduce Querying
*MapReduce* is a programming model for processing large amounts of data in bulk across many machines, popularizes by Google

A limited form of MapReduce is supported by some NoSQL datastores, including MongoDB and CouchDB as a mechanism for performing read-only queries across many documents

MapReduce is neither a declarative query language nor a fully imperative query API, but somewhere in between: the logic of the query is expressed with snippets of code, which are called repeatedly by the processing framework

It is based on `map` (also known as `collect`) and `reduce` (also known as `fold` or `inject`) functions that exist in many functional programming languages

Say you are a marine biologist and you add an observation record to your database every time you see animals in the ocean, now you want to generate a report saying how many sharks you have sighted per month

In PostgreSQL you might express that query like this
```SQL
SELECT date_trunc('month', observation_timestamp) AS observation_month,
    sum(num_animals) AS total_animals
FROM observations
WHERE family = 'Sharks'
GROUP BY observation_month;
```
The `date_trunc('month', timestamp)` function determines the calendar month containing `timestamp` and returns another timestamp representing the beginning of that month, in other words it rounds a timestamp down to the nearest month

1. This query first filters the observations to only show specifies in the `Sharks` family
2. Then it groups the observations by the calendar month in which they occurred
3. It adds up the number of animals seen in all observations in that month

This can be expressed with MongoDB's MapReduce feature as follows:
```javascript
db.observations.mapReduce(
    function map() {
        var year = this.observationTimestamp.getFullYear();
        var month = this.observationTimestamp.getMonth() + 1;
        emit(year + "-" + month, this.numAnimals);
    },
        function reduce(key, values) {
        return Array.sum(values);
    },
    {
        query: { family: "Sharks" },
        out: "monthlySharkReport"
    }
);
```
1. The filter to consider only shark species can be specified declaratively (this is a MongoDB-specific extension to MapReduce)
2. The JavaScript function `map` is called once for every document that matches `query`, with `this` set to the document object
3. The `map` function emits a key (a string consisting of year and month, such as "2013-12" or "2014-1") and a value (the number of animals in that observations)
4. The key-value pairs emitted by `map` are grouped by key, for all key-value pairs with the same key, the `reduce` function is called once
5. The `reduce` function adds up the number of animals from all observations in a particular month
6. The final output is written to the collection `monthlySharkReports`

The `map` and `reduce` functions are somewhat restricted in what they are allowed to do

First, they must be *pure* functions, which means they only use the data that is passed to them as input, they cannot perform additional database queries, and they **must not** have any side effects

*side effects are any observable change in the state of the system that occurs as a result of executing a function or expression beyond just returning a value*

These restrictions allow the database to run the functions anywhere, in any order, and rerun them on failure, however, they are nevertheless powerful: they can parse strings, call library functions, perform calculations, and more

A usability problem with MapReduce is that you have to write two carefully coordinated JavaScript functions, which is often harder than writing a single query, moreover, a declarative query language offers more opportunities for a query optimizer to improve the performance of a query

For these reasons, MongoDB 2.2 added support for a declarative query language called the **aggregation pipeline** in this language, the same shark-counting query looks like this

```javascript
db.observations.aggregate([
    { $match: { family: "Sharks" }},
    { $group: {
        _id: {
            year: { $year: "$observationTimestamp" }.
            month: { $month: "observationTimestamp" }
        },
        totalAnimals: { $sum: "$numAnimals"}
    }}
])
```

The aggregation pipeline language is similar in expressiveness to a subset of SQL, but it uses a JSON-based syntax rather than SQL's English-sentence-style syntax

Moral of the story is that a NoSQL system may find itself accidentally reinventing SQL in disguise 

### My Explanation on MapReduce
My explanation of MapReduce: It's basically a programming model designed to process large datasets by breaking the task into two distinct phases:
1. Map phase: The purpose is to process input data and transform it into a set of intermediate key-value pairs, each item is independently processed by a "map" function, this function `emits` key-value pairs based on the logic you provide

The framework then groups all values by their keys, this means each key becomes associated with a **list** of values whom share that key

2. Reduce phase: The purpose is to aggregate or summarize the intermediate key-value pairs produced by the map function, for each unique key the "reduce" function receives an array of values and combined them (e.g summing, averaging, etc) into a single output

It is important to note that the "reduce" steps aggregates all the value associated with a key into one final value for that key, but you can have many keys overall

A few reduce functions include sum, average, count occurrences, and max/min value