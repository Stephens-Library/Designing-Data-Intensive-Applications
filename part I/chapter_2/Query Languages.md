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