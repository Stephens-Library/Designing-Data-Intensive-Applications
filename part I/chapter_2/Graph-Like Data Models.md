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