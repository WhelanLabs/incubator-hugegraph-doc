---
title: "HugeGraph Examples"
linkTitle: "HugeGraph Examples"
weight: 2
---

### 1 Overview

This example uses the [TitanDB Getting Started](http://s3.thinkaurelius.com/docs/titan/1.0.0/getting-started.html) as a template to demonstrate how to use HugeGraph and compare the differences between HugeGraph and TitanDB.

#### 1.1 Similarities and Differences between HugeGraph and TitanDB

Both HugeGraph and TitanDB are graph databases based on the [Apache TinkerPop3](https://tinkerpop.apache.org) framework, both support the [Gremlin](https://tinkerpop.apache.org/gremlin.html) graph query language, and have many similarities in usage methods and interfaces. However, HugeGraph is newly designed and developed. Its code structure is clear, its functions are richer, and its interface is more friendly.

Compared to TitanDB, the main characteristics of HugeGraph are as follows:

- HugeGraph currently has a complete set of tool components including HugeGraph-API, HugeGraph-Client, HugeGraph-Loader, HugeGraph-Studio, HugeGraph-Spark, etc., which can accomplish system integration, data loading, graph visualization querying, Spark connection, and other functions.
- HugeGraph has the concept of Server and Client, and third-party systems can access it through various methods such as jar reference, client, API, etc., while TitanDB only supports jar reference access.
- HugeGraph's schema needs to be explicitly defined, and all insertions and queries need to go through strict schema validation. Currently, implicit schema creation is not supported.
- HugeGraph fully utilizes the characteristics of the backend storage system to achieve efficient data storage and access, while TitanDB ignores backend differences with a unified Kv structure.
- HugeGraph's update operations can be performed on demand (e.g. updating a specific attribute), which performs better in terms of performance. TitanDB's updates are read and update operations.
- HugeGraph's VertexId and EdgeId both support concatenation, which can achieve automatic deduplication and better query performance. All Ids in TitanDB are auto-generated, and querying requires indexing.

#### 1.2 Character Relationship Graph

This example uses the Property Graph Model to describe the relationships between characters in Greek mythology, also known as the Character Relationship Graph. The specific relationships are detailed in the following figure.

<div style="text-align: center;">
  <img src="/docs/images/graph-of-gods.png" alt="image">
</div>

In this graph, circular nodes represent entities (Vertices), arrows represent relationships (Edges), and the contents of the boxes are properties.

There are two types of vertices in this relationship graph, characters and locations, as shown in the following table:

| Name        | Type      | Properties          |
|-----------|--------|---------------|
| character | vertex | name,age,type |
| location  | vertex | name          |

There are six types of relationships, namely father, mother, brother, battled, lives, and pet. The specific information about the relationship graph is as follows:

| 名称      | 类型   | source vertex label | target vertex label | 属性     |
|---------|------|---------------------|---------------------|--------|
| father  | edge | character           | character           | -      |
| mother  | edge | character           | character           | -      |
| brother | edge | character           | character           | -      |
| pet     | edge | character           | character           | -      |
| lives   | edge | character           | location            | reason |

In HugeGraph, each edge label can only act on a pair of source vertex label and target vertex label. That is to say, if a relationship father is defined in a graph to connect character and character, then farther cannot connect other vertex labels.

Therefore, in this example, monster, god, human, and demigod in the original TitanDB are all represented by the vertex label `character`, and the attribute type is added to identify the type of character.  The `edge labels` remain the same as in the original TitanDB to satisfy the constraint of `edge label` uniqueness. Of course, adjusting the `name` of the `edge label` can also achieve this.

### 2 Graph Schema and Data Ingest Examples

HugeGraph needs to display and create a Schema, so it needs to create PropertyKey, VertexLabel, EdgeLabel in turn, and IndexLabel needs to be created if there is an index.

#### 2.1 Graph Schema

```groovy
schema = hugegraph.schema()

schema.propertyKey("name").asText().ifNotExist().create()
schema.propertyKey("age").asInt().ifNotExist().create()
schema.propertyKey("time").asInt().ifNotExist().create()
schema.propertyKey("reason").asText().ifNotExist().create()
schema.propertyKey("type").asText().ifNotExist().create()

schema.vertexLabel("character").properties("name", "age", "type").primaryKeys("name").nullableKeys("age").ifNotExist().create()
schema.vertexLabel("location").properties("name").primaryKeys("name").ifNotExist().create()

schema.edgeLabel("father").link("character", "character").ifNotExist().create()
schema.edgeLabel("mother").link("character", "character").ifNotExist().create()
schema.edgeLabel("battled").link("character", "character").properties("time").ifNotExist().create()
schema.edgeLabel("lives").link("character", "location").properties("reason").nullableKeys("reason").ifNotExist().create()
schema.edgeLabel("pet").link("character", "character").ifNotExist().create()
schema.edgeLabel("brother").link("character", "character").ifNotExist().create()
```

#### 2.2 Graph Data

```groovy
// add vertices
Vertex saturn = graph.addVertex(T.label, "character", "name", "saturn", "age", 10000, "type", "titan")
Vertex sky = graph.addVertex(T.label, "location", "name", "sky")
Vertex sea = graph.addVertex(T.label, "location", "name", "sea")
Vertex jupiter = graph.addVertex(T.label, "character", "name", "jupiter", "age", 5000, "type", "god")
Vertex neptune = graph.addVertex(T.label, "character", "name", "neptune", "age", 4500, "type", "god")
Vertex hercules = graph.addVertex(T.label, "character", "name", "hercules", "age", 30, "type", "demigod")
Vertex alcmene = graph.addVertex(T.label, "character", "name", "alcmene", "age", 45, "type", "human")
Vertex pluto = graph.addVertex(T.label, "character", "name", "pluto", "age", 4000, "type", "god")
Vertex nemean = graph.addVertex(T.label, "character", "name", "nemean", "type", "monster")
Vertex hydra = graph.addVertex(T.label, "character", "name", "hydra", "type", "monster")
Vertex cerberus = graph.addVertex(T.label, "character", "name", "cerberus", "type", "monster")
Vertex tartarus = graph.addVertex(T.label, "location", "name", "tartarus")

// add edges
jupiter.addEdge("father", saturn)
jupiter.addEdge("lives", sky, "reason", "loves fresh breezes")
jupiter.addEdge("brother", neptune)
jupiter.addEdge("brother", pluto)
neptune.addEdge("lives", sea, "reason", "loves waves")
neptune.addEdge("brother", jupiter)
neptune.addEdge("brother", pluto)
hercules.addEdge("father", jupiter)
hercules.addEdge("mother", alcmene)
hercules.addEdge("battled", nemean, "time", 1)
hercules.addEdge("battled", hydra, "time", 2)
hercules.addEdge("battled", cerberus, "time", 12)
pluto.addEdge("brother", jupiter)
pluto.addEdge("brother", neptune)
pluto.addEdge("lives", tartarus, "reason", "no fear of death")
pluto.addEdge("pet", cerberus)
cerberus.addEdge("lives", tartarus)
```

#### 2.3 Indices

In HugeGraph, Ids are generated automatically by default. However, if the user specifies a list of fields as `primaryKeys` for a `VertexLabel`, the Id strategy for the `VertexLabel` will be automatically switched to the `primaryKeys` strategy. After enabling the `primaryKeys` strategy, HugeGraph generates the `VertexId` by concatenating `vertexLabel+primaryKeys`, which can achieve automatic deduplication, and the properties in `primaryKeys` can be used for fast query without the need to create additional indexes. For example, if both "character" and "location" have a `primaryKeys("name")` attribute, vertices can be queried without creating additional indexes using `g.V().hasLabel('character') .has('name','hercules')`.

### 3 Graph Traversal Examples

#### 3.1 Traversal Query

**1\. Find the grandfather of hercules**

```groovy
g.V().hasLabel('character').has('name','hercules').out('father').out('father')
```

This can also be done using `repeat`:

```groovy
g.V().hasLabel('character').has('name','hercules').repeat(__.out('father')).times(2)
```

**2\. Find the name of hercules's father**

```groovy
g.V().hasLabel('character').has('name','hercules').out('father').value('name')
```

**3\. Find the characters with age > 100**

```groovy
g.V().hasLabel('character').has('age',gt(100))
```

**4\. Find who are pluto's cohabitants**

```groovy
g.V().hasLabel('character').has('name','pluto').out('lives').in('lives').values('name')
```

**5\. Find pluto can't be his own cohabitant**

```groovy
pluto = g.V().hasLabel('character').has('name', 'pluto')
g.V(pluto).out('lives').in('lives').where(is(neq(pluto)).values('name')

// use 'as'
g.V().hasLabel('character').has('name', 'pluto').as('x').out('lives').in('lives').where(neq('x')).values('name')
```

**6\. Pluto's Brothers**

```groovy
pluto = g.V().hasLabel('character').has('name', 'pluto').next()
// where do pluto's brothers live?
g.V(pluto).out('brother').out('lives').values('name')

// which brother lives in which place?
g.V(pluto).out('brother').as('god').out('lives').as('place').select('god','place')

// what is the name of the brother and the name of the place?
g.V(pluto).out('brother').as('god').out('lives').as('place').select('god','place').by('name')
```

It is recommended to use [HugeGraph-Studio](/docs/quickstart/hugegraph-studio) to execute the above code in a visual way. In addition, the above code can also be executed in various ways such as HugeGraph-Client, HugeApi, GremlinConsole, and GremlinDriver.

#### 3.2 Summary

HugeGraph currently supports the `Gremlin` query language, and users can use the Gremlin query language or the REST API to perform various types of queries and operations on the graph.


