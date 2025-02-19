# Graph-Like Data Models
We saw earlier that many-to-many relationships are an important distinguishing feature between different data models

If your application has mostly one-to-many relationships (tree-structured data) or no relationships between records, the document model is appropriate

But what if many-to-many relationships are very common in your data? The relational model can handle simple cases of many-to-many relationships, but as the connections within your data become more complex, it becomes more natural to start modeling your data as a graph

A graph consists of two kinds of objects: *vertices* (also known as *nodes* or *entities*) and *edges* (also known as *relationships* or *arcs*)

Many kinds of data can be modeled as a graph, typical examples include:
- **Social graphs**: Vertices are people and edges indicate which people know each other
- **The web graph**: Vertices are web pages and edges indicate HTML links to other pages
- **Road or rail Networks**: Vertices are junctions, and edges represent the roads or railway lines between them

Well-known algorithms can operate on these graphs: for example, car navigation systems search for the shortest path between two points in a road network and PageRank can be used on the web graph to determine the popularity of web pages and thus its ranking in search results

In this examples above all the vertices in the graph are *homogeneous*, that is all vertices in the graph represent the same thing (people, web pages, or road junctions respectively)

However, graphs can also store non-homogenous data, for example Facebook maintains a single graph with many different types of vertices and edges: vertices represent people, locations, events, check-ins, and comments made by users while edges indicate which people are friends, which check-in happened in which location, who commented on which post, etc

![image](<photos/non-homogenous_graph.png>)
*Example of graph-structured data (boxes represent vertices and arrows represent edges)*

In the figure above, it shows two people Lucy from Idaho and Alain from Beaune, France they are married and living in London, there are several different, but related ways of structuring and querying data in graphs

In this section we will discuss the *property graph* model (implemented by Neo4j, Titan, and InfiniteGraph) and the *triple-store* model (implemented by Datomic, AllegroGraph, and others)

We will look at three declarative query languages for graphs: Cypher, SPARQL, and Datalog, besides these there are also imperative graph query languages such as Gremlin and graph processing frameworks like Pregel

## Property Graphs
In the property graph model, each vertex consists of: 
- A unique identifier
- A set of outgoing edges
- A set of incoming edges
- A collection of properties (key-value pairs)

Each edge consists of:
- A unique identifier
- The vertex at which the edge starts (the *tail* vertex)
- The vertex at which the edge ends (the *head* vertex)
- A label to describe the kind of relationship between the two vertices
- A collection of properties (key-value pairs)

You can think of a graph store as consisting of two relational tables, one for vertices and one for edges

The head and tail vertex are stored for each edge; if you want the set of incoming or outgoing edges for a vertex, you can query the `edges` table by `head_vertex` or `tail_vertex` respectively

```sql
CREATE TABLE vertices (
    vertex_id integer PRIMARY KEY,
    properties json
);
CREATE TABLE edges (
    edge_id integer PRIMARY KEY,
    tail_vertex integer REFERENCES vertices (vertex_id),
    head_vertex integer REFERENCES vertices (vertex_id),
    label text,
    properties json
);

CREATE INDEX edges_tails ON edges (tail_vertex);
CREATE INDEX edges_heads ON edges (head_vertex);
```
*This figure represents a property graph using a relational schema*

1. Any vertex can have an edge connecting it with any other vertex, there is no schema that restricts which kinds of things can or cannot be associated
2. Given any vertex, you can efficiently find both its incoming and outgoing edges, and thus *traverse* the graph (which is why we create indexes on both the `tail_vertex` and the `head_vertex`)
3. By using different labels for different kinds of relationships, you can store several different kinds of information in a single graph while still maintaining a clean data model

Graphs are good for evolvability, as you add features to your application, a graph can easily be extended to accommodate changes in your application's data structures

## The Cypher Query Language
*Cypher* is a declarative query language for property graphs, created for the *Neo4j* graph database
```sql
CREATE
    (NAmerica:Location {name:'North America', type:'continent'}),
    (USA:Location {name:'United States', type:'country' }),
    (Idaho:Location {name:'Idaho', type:'state' }),
    (Lucy:Person {name:'Lucy' }),
    (Idaho) -[:WITHIN]-> (USA) -[:WITHIN]-> (NAmerica),
    (Lucy) -[:BORN_IN]-> (Idaho)
```
This shows the Cypher query language to insert the left hand portion of the non-homogenous graph above into a graph database
![image](<photos/non-homogenous_graph.png
Each vertex is given a symbolic name like `USA` or `Idaho` and other parts of the query can use those names to create edges between the vertices, using an arrow notation: `(Idaho) -[:WITHIN]-> (USA)` creates an edge labeled `WITHIN`, with `Idaho` as the tail node and `USA` as the head node

When all the vertices and edges are added to the database, we can start asking interesting questions: for example, *find the names of all the people who emigrated from the United States to Europe*

To be more precise, here we want to find all the vertices that have a `BORN_IN` edge to a location within the `US` but also a `LIVING_IN` edge to a location within Europe, and return the `name` property of each of those vertices

Below is the following Cypher query:
```sql
MATCH
    (person) -[:BORN_IN]-> () -[:WITHIN*0..]-> (us:Location {name:'United States'}),
    (person) -[:LIVES_IN]-> () -[:WITHIN*0..]-> (eu:Location {name:'Europe'})
RETURN person.name
```
Here the same arrow notation is used in a `MATCH` clause to find patterns in the graph

The query can be read as follows:
Find any vertex that meets *both* of the following conditions
1. `person` has an outgoing `BORN_IN` edge to some vertex, from that vertex you can follow a chain of outgoing `WITHIN` edges until eventually you reach a vertex of type `Location`, whose `name` property is equal to `United States`
2. That same `person` vertex also has an outgoing `LIVES_IN` edge following that edge and then a chain of outgoing `WITHIN` edges, you eventually reach a vertex of type `lLocation` whose `name` property is equal to "Europe"

For each such `person` vertex, return the `name` property

*note: \*0.. means 0 or more hops, the number of hops is the number of jumps you have to make in order to get to the specified data, 0 hopes for example would mean no relationship is traversed at all and the node you're already at must be a `Location` with the name `United States` there's no next node*

There are several possible ways of executing the query, the description given here suggests that you start by scanning all the people in the database, examine each person's birthplace and residence, and return only those people who meet the criteria

But equivalently, you could start with the two `Location` vertices and work backward, if there is an index on the `name` property, you can probably efficiently find the two vertices representing the US and Europe, then you can proceed to find all locations (states, regions, cities, etc) in the US and Europe respectively by following all incoming `WITHIN` edges, and finally you can look for people who can be found through incoming `BORN_IN` or `LIVES_IN` edge at one of the location vertices

As is typical for a declarative query language, you don't need to specify such execution details when writing the query: the query optimizer automatically chooses the strategy that is predicted to be the most efficient, so you can get on with writing the rest of your application

## Graph Queries in SQL
As seen above where re represented the graph data with a relational schema one might ask, can we also query it using SQL?

The answer is yes, we can, but with difficulty. In a relational database, you usually know in advance which joins you need in your query, in a graph query you may need to traverse a variable number of edges before you find the vertex you're looking for, that is the number of joins is not fixed in advance

Hence we use the Cypher operator `:WITHIN*0..` which expresses very concisely follow a `WITHIN` edge zero or more times, similar to `*` in a regex, this is because `LIVES_IN` may point to a city that points to a state within the US or it could point to the US directly

Since SQL:1999 this idea of variable-length traversal paths in a query can be expressed using something called `recursive common table expressions` (the `WITH RECURSIVE` syntax), below is the same query in SQL, but the syntax is clumsy compared to Cypher
```sql
WITH RECURSIVE
    -- in_usa is the set of vertex IDs of all locations within the United States
    in_usa(vertex_id) AS (
        SELECT vertex_id FROM vertices WHERE properties->>'name' = 'United States'
        UNION
        SELECT edges.tail_vertex FROM edges
        JOIN in_usa ON edges.head_vertex = in_usa.vertex_id
        WHERE edges.label = 'within'
    ),
    -- in_europe is the set of vertex IDs of all locations within Europe
    in_europe(vertex_id) AS (
        SELECT vertex_id FROM vertices WHERE properties->>'name' = 'Europe'
        UNION
        SELECT edges.tail_vertex FROM edges
        JOIN in_europe ON edges.head_vertex = in_europe.vertex_id
        WHERE edges.label = 'within'
    ),
    -- born_in_usa is the set of vertex IDs of all people born in the US
    born_in_usa(vertex_id) AS (
        SELECT edges.tail_vertex FROM edges
        JOIN in_usa ON edges.head_vertex = in_usa.vertex_id
        WHERE edges.label = 'born_in'
    ),
    -- lives_in_europe is the set of vertex IDs of all people living in Europe
        lives_in_europe(vertex_id) AS (
        SELECT edges.tail_vertex FROM edges
        JOIN in_europe ON edges.head_vertex = in_europe.vertex_id
        WHERE edges.label = 'lives_in'
    )

SELECT vertices.properties->>'name'
FROM vertices
-- join to find those people who were both born in the US *and* live in Europe
JOIN born_in_usa ON vertices.vertex_id = born_in_usa.vertex_id
JOIN lives_in_europe ON vertices.vertex_id = lives_in_europe.vertex_id;
```

1. First find the vertex whose `name` property has the value `"United States"`, and make it the first element of the set of vertices `in_usa`
2. Follow all incoming `within` edges from `vertices` in the set `in_usa` and add them to the same set, until all incoming `within` edges have been visited
3. Do the same starting with the vertex whose `name` property has the value `"Europe"` and build up the set of vertices `in_europe`
4. For each of the vertices in the set `in_usa` follow incoming `born_in` edges to find people who were born in some place within the United States
5. For each of the vertices in the set `in_europe` follow incoming `lives_in` edges to find people who live in Europe
6. Intersect the set of people born in the USA with the set of people living in Europe by joining them

If the same query can be written in 4 lines in one query language but requires 29 lines in another, that just shows that different data models are designed to satisfy different use cases

