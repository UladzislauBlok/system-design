# Chapter 2.3 Relational Model Versus Document Model: Graph-Like Data Models

The document model is suitable for data with mostly one-to-many (tree-structured) or no relationships. When many-to-many relationships become common and complex, it's more natural to use a graph model.

A graph consists of:

- Vertices (or nodes / entities)
- Edges (or relationships / arcs)

Graphs can model diverse data, often heterogeneously:

- Social Graphs: People (vertices) and their connections (edges).
- Web Graph: Web pages (vertices) and HTML links (edges).
- Road Networks: Junctions (vertices) and roads (edges).

Graph Algorithms and Models:

- Algorithms: Well-known algorithms operate on graphs, such as finding the shortest path (e.g., in navigation) or PageRank (for web page popularity).
- Models: Key graph models include:
  - Property Graph Model (used by Neo4j, Titan)
  - Triple-Store Model (used by Datomic, AllegroGraph)

- Query Languages: Data in graphs can be queried using declarative languages like Cypher, SPARQL, and Datalog, as well as imperative languages like Gremlin.

<br>

---

<br>

### Property Graphs

In the property graph model, each vertex consists of:

- A unique identifier
- A set of outgoing edges
- A set of incoming edges
- A collection of properties (key-value pairs)

Each edge consists of:

- A unique identifier
- The vertex at which the edge starts (the tail vertex)
- The vertex at which the edge ends (the head vertex)
- A label to describe the kind of relationship between the two vertices
- A collection of properties (key-value pairs)

![graph-db](./images/graph-structured-data.png)

Representing a property graph using a relational schema:

```
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

Some important aspects of this model are:

1. Any vertex can have an edge connecting it with any other vertex. There is no schema that restricts which kinds of things can or cannot be associated.
2. Given any vertex, you can efficiently find both its incoming and its outgoing edges, and thus traverse the graph—i.e., follow a path through a chain of vertices - both forward and backward.
3. By using different labels for different kinds of relationships, you can store several different kinds of information in a single graph, while still maintaining a clean data model.

Graph data models offer great flexibility compared to traditional relational schemas:

- Handling Complexity: Graphs can easily express structures that are difficult in relational models, such as:
  - Varying Regional Structures: Modeling different country divisions (e.g., French départements vs. US counties).
  - Historical/Political Quirks: Representing complex relationships like a "country within a country."
  - Granularity: Allowing different levels of detail for related data (e.g., current residence specified as a city, birthplace as a state).

- Evolvability: Graphs are excellent for accommodating changes and new features:
  - You can easily extend the graph to include new types of information and relationships (e.g., adding vertices for food allergens and edges linking people, allergens, and foods).
  - This allows for complex queries, such as finding out which foods are safe to eat for a person with specific allergies.

<br>

---

<br>

### The Cypher Query Language

Cypher is a declarative query language for property graphs, created for the Neo4j graph database.

Cypher query to insert the lefthand portion of graph above:

```
CREATE
  (NAmerica:Location {name:'North America', type:'continent'}),
  (USA:Location {name:'United States', type:'country' }),
  (Idaho:Location {name:'Idaho', type:'state' }),
  (Lucy:Person {name:'Lucy' }),
  (Idaho) -[:WITHIN]-> (USA) -[:WITHIN]-> (NAmerica),
  (Lucy) -[:BORN_IN]-> (Idaho)
```

- Data is added to a graph by defining vertices (nodes) and edges (relationships).
- Vertices are given symbolic names (e.g., USA or Idaho).
- Edges are created between these named vertices using an arrow notation.

Query in Cypher to find the names of all the people who emigrated from the United States to Europe:

```
MATCH
  (person) -[:BORN_IN]-> () -[:WITHIN*0..]-> (us:Location {name:'United States'}),
  (person) -[:LIVES_IN]-> () -[:WITHIN*0..]-> (eu:Location {name:'Europe'})
RETURN person.name

```

The query can be read as follows:

- Find any vertex (call it person) that meets both of the following conditions:
  1. person has an outgoing BORN_IN edge to some vertex. From that vertex, you can follow a chain of outgoing WITHIN edges until eventually you reach a vertex of type Location, whose name property is equal to "United States".
  2. That same person vertex also has an outgoing LIVES_IN edge. Following that edge, and then a chain of outgoing WITHIN edges, you eventually reach a vertex of type Location, whose name property is equal to "Europe".
- For each such person vertex, return the name property.

When executing a complex graph query, the system has multiple strategies:

1. Scanning Forward (Person-Centric): Start by scanning all people in the database, then examine each person's BORN_IN and LIVING_IN edges to check if the destinations meet the US/Europe criteria.
2. Working Backward (Location-Centric):
   - Find the key location nodes (US and Europe) using an index.
   - Traverse backward along incoming WITHIN edges to find all lower-level locations (states, cities, etc.) within the US and Europe.
   - Finally, look for people connected to these lower-level locations via incoming BORN_IN or LIVES_IN edges.

Declarative Advantage: As this is a declarative query, you only specify what you want (the results), not how to get them. The query optimizer automatically selects the most efficient execution strategy.

### Graph Queries in SQL

While graph data can be stored in a relational structure, querying it with SQL presents challenges, particularly when dealing with variable-length traversal paths.

The Difficulty in SQL

- In typical relational queries, the number of joins is known in advance.
- In graph queries, you often need to traverse a variable number of edges to reach the target vertex.

Previous Example: Location Hierarchy

- A LIVES_IN edge might point directly to a country, or to a city, which is then WITHIN a state, which is WITHIN a country.
- Graph query languages handle this concisely: In Cypher, the syntax [:WITHIN*0..] means "follow the WITHIN edge zero or more times," allowing for variable path lengths.

SQL Solution

Since SQL:1999, the concept of variable-length path traversal can be implemented using Recursive Common Table Expressions (CTE), typically using the WITH RECURSIVE syntax.

- This technique allows the query to repeatedly join the table to itself until a condition is met.
- Conclusion: Although technically possible in SQL, this syntax is often clumsy and less intuitive compared to specialized graph query languages like Cypher.

```
  WITH RECURSIVE
-- in_usa is the set of vertex IDs of all locations within the United States
in_usa(vertex_id) AS (
    SELECT vertex_id FROM vertices WHERE properties->>'name' = 'United States' 1
  UNION
    SELECT edges.tail_vertex FROM edges 2
      JOIN in_usa ON edges.head_vertex = in_usa.vertex_id
      WHERE edges.label = 'within'
),

-- in_europe is the set of vertex IDs of all locations within Europe
in_europe(vertex_id) AS (
    SELECT vertex_id FROM vertices WHERE properties->>'name' = 'Europe' 3
  UNION
    SELECT edges.tail_vertex FROM edges
      JOIN in_europe ON edges.head_vertex = in_europe.vertex_id
      WHERE edges.label = 'within'
),

-- born_in_usa is the set of vertex IDs of all people born in the US
born_in_usa(vertex_id) AS ( 4
  SELECT edges.tail_vertex FROM edges
    JOIN in_usa ON edges.head_vertex = in_usa.vertex_id
    WHERE edges.label = 'born_in'
),

- lives_in_europe is the set of vertex IDs of all people living in Europe
lives_in_europe(vertex_id) AS ( 5
  SELECT edges.tail_vertex FROM edges
    JOIN in_europe ON edges.head_vertex = in_europe.vertex_id
    WHERE edges.label = 'lives_in'
)

SELECT vertices.properties->>'name'
FROM vertices
-- join to find those people who were both born in the US *and* live in Europe
JOIN born_in_usa ON vertices.vertex_id = born_in_usa.vertex_id 6
JOIN lives_in_europe ON vertices.vertex_id = lives_in_europe.vertex_id;
```

1. First find the vertex whose name property has the value "United States", and make it the first element of the set of vertices in_usa.
2. Follow all incoming within edges from vertices in the set in_usa, and add them to the same set, until all incoming within edges have been visited.
3. Do the same starting with the vertex whose name property has the value "Europe", and build up the set of vertices in_europe.
4. For each of the vertices in the set in_usa, follow incoming born_in edges to find people who were born in some place within the United States.
5. Similarly, for each of the vertices in the set in_europe, follow incoming lives_in edges to find people who live in Europe.
6. Finally, intersect the set of people born in the USA with the set of people living in Europe, by joining them.

If the same query can be written in 4 lines in one query language but requires 29 lines in another, that just shows that different data models are designed to satisfy different use cases. It’s important to pick a data model that is suitable for your application.

<br>

---

<br>

### Triple-Stores and SPARQL

The triple-store model is largely equivalent to the property graph model, differing mainly in terminology. However, it's worth understanding because various tools and languages exist for triple-stores that can enhance application development.

In this model, all data is stored as simple three-part statements called triples:

`(subject, predicate, object)`

Example: `(Jim, likes, bananas)`

- **_Jim_** is the subject.
- **_likes_** is the predicate (the verb/relationship).
- **_bananas_** is the object.

The elements of a triple map to a graph as follows:

- Subject: Always equivalent to a vertex (node) in the graph.
- Object: Can be one of two things:
  1. A primitive value (e.g., string, number):
     - The (predicate, object) pair is equivalent to a property key and value on the subject vertex.
     - Example: (lucy, age, 33) is like a vertex lucy with the property {"age":33}.
  2. Another vertex in the graph:
     - The predicate is equivalent to an edge (relationship) in the graph.
     - The subject is the tail vertex, and the object is the head vertex.
     - Example: (lucy, marriedTo, alain). Both lucy and alain are vertices, and marriedTo is the edge connecting them.

The code below shows the same data as in example above, written as triples in a format called Turtle, a subset of Notation3 (N3):

```
@prefix : <urn:example:>.
_:lucy a :Person.
_:lucy :name "Lucy".
_:lucy :bornIn _:idaho.
_:idaho a :Location.
_:idaho :name "Idaho".
_:idaho :type "state".
_:idaho :within _:usa.
_:usa a :Location.
_:usa :name "United States".
_:usa :type "country".
_:usa :within _:namerica.
_:namerica a :Location.
_:namerica :name "North America".
_:namerica :type "continent".
```

In triple-store data formats (like Turtle), vertices are often represented using identifiers like \_:someName. These names are only locally significant within the file, ensuring that related triples refer to the same specific vertex.

- When the predicate is an edge, the object is a vertex:
  - Example: _:idaho :within _:usa
- When the predicate is a property, the object is a string literal (value):
  - Example: \_:usa :name "United States"

To avoid repeating the subject for every triple, formats like Turtle allow you to use semicolons (;) to list multiple statements about the same subject, making the data more concise and readable.

```
  @prefix : <urn:example:>.
_:lucy a :Person; :name "Lucy"; :bornIn _:idaho.
_:idaho a :Location; :name "Idaho"; :type "state"; :within _:usa.
_:usa a :Location; :name "United States"; :type "country"; :within _:namerica.
_:namerica a :Location; :name "North America"; :type "continent".
```

_The SPARQL query language_

SPARQL is a query language for triple-stores using the RDF data model. (It is an acronym for SPARQL Protocol and RDF Query Language, pronounced “sparkle.”) It predates Cypher, and since Cypher’s pattern matching is borrowed from SPARQL, they look quite similar.

The same query as before—finding people who have moved from the US to Europe — is even more concise in SPARQL than it is in Cypher

```
PREFIX : <urn:example:>

SELECT ?personName WHERE {
  ?person :name ?personName.
  ?person :bornIn / :within* / :name "United States".
  ?person :livesIn / :within* / :name "Europe".
}
```

The structure is very similar. The following two expressions are equivalent (variables start with a question mark in SPARQL):

```
(person) -[:BORN_IN]-> () -[:WITHIN*0..]-> (location) # Cypher

?person :bornIn / :within* ?location. # SPARQL
```

In the following expression, the variable usa is bound to any vertex that has a name property whose value is the string "United States":

```
(usa {name:'United States'}) # Cypher

?usa :name "United States". # SPARQL
```
