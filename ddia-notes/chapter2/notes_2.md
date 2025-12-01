# Chapter 2.2 Relational Model Versus Document Model: Query Languages for Data

The introduction of SQL and the relational model brought a shift from imperative to declarative querying. Imperative languages require specifying the exact sequence of steps (how) to retrieve data, much like stepping through code line by line. Conversely, a declarative language like SQL only requires you to specify the desired data pattern (what) you want, leaving the database's query optimizer to independently determine the most efficient execution plan (indexes, join order, etc.).

This declarative approach is beneficial because it is more concise and, more importantly, it allows the database system to perform automatic optimizations and implement parallel execution freely, without breaking queries. Since the user doesn't dictate the algorithm, the database can safely update its internal methods for better performance without requiring changes to the application code.

<br>

---

<br>

### MapReduce Querying

MapReduce is a programming model, popularized by Google, for processing large datasets in bulk across many machines. Some NoSQL databases, like MongoDB and CouchDB, support a limited version of MapReduce for read-only queries.

MapReduce is an intermediate approach between declarative query languages and imperative APIs, where query logic is expressed using repeatable code snippets based on the map (or collect) and reduce (or fold) functional programming concepts.i

**PostgreSQL (Declarative SQL)**

This is done using declarative SQL with GROUP BY and an aggregation function:

```
SELECT date_trunc('month', observation_timestamp) AS observation_month,
       sum(num_animals) AS total_animals
FROM observations
WHERE family = 'Sharks'
GROUP BY observation_month;
```

**MongoDB MapReduce (Code Snippets)**

In MongoDB, this is achieved using the mapReduce function:

1. The query: { family: "Sharks" } filters the documents (a MongoDB-specific extension).
2. The map function runs on each matching document, extracting the year/month and the number of animals. It emits a key (the month, e.g., "2013-12") and a value (the animal count).
3. The emitted key-value pairs are grouped by key (month).
4. The reduce function is called for each unique key with a list of all associated values, and it aggregates (sums) them.
5. The final result is stored in the monthlySharkReport collection.

```
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

For example, say the observations collection contains these two documents:

```
{
    observationTimestamp: Date.parse("Mon, 25 Dec 1995 12:34:56 GMT"),
    family: "Sharks",
    species: "Carcharodon carcharias",
    numAnimals: 3
}
{
    observationTimestamp: Date.parse("Tue, 12 Dec 1995 16:17:18 GMT"),
    family: "Sharks",
    species: "Carcharias taurus",
    numAnimals: 4
}
```

The MapReduce process is illustrated by the example:

- Map runs once per document: emit("1995-12", 3) and emit("1995-12", 4).
- Reduce runs on the grouped data: reduce("1995-12", [3, 4]), returning 7.

##### Purity and Execution

The map and reduce functions must be pure functions:

- They only use their input data.
- They cannot perform additional database queries or have side effects.

This restriction allows the database to run the functions anywhere, in any order, and retry them on failure. Despite the constraints, they can perform complex tasks like parsing and calculations.

##### Context and Usability

- MapReduce is a low-level programming model for distributed execution, but it's not the only one; many distributed SQL implementations exist that don't use MapReduce.
- The ability to embed code (like JavaScript) in a query is powerful but not unique to MapReduce; some SQL databases offer similar extensions.
- Usability Issue: Writing two coordinated JavaScript functions (map and reduce) is often harder than writing a single declarative query, and it offers fewer opportunities for query optimization.

To address the usability and optimization issues of MapReduce, MongoDB introduced the Aggregation Pipeline (in version 2.2), a declarative query language.

The shark-counting query in this pipeline looks like this:

```
b.observations.aggregate([
    { $match: { family: "Sharks" } },
    { $group: {
        _id: {
            year: { $year: "$observationTimestamp" },
            month: { $month: "$observationTimestamp" }
        },
        totalAnimals: { $sum: "$numAnimals" }
    } }
]);
```

The aggregation pipeline is similar to a subset of SQL in expressiveness but uses a JSON-based syntax. This shows that NoSQL systems sometimes find themselves reinventing SQL concepts, even if the syntax is different.
