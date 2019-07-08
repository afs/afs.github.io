---
layout: doc
title: Extending SHACL to RDF Datasets
---
<div class="docinfo">
     <p>Andy Seaborne</p>
</div>

This note describes applying the 
[SHACL Shapes Constraint Language](https://www.w3.org/TR/shacl/)
to [RDF Datasets](https://www.w3.org/TR/rdf11-concepts/#section-dataset).

SHACL defines a set of validation conditions in a [Shapes
Graph](https://www.w3.org/TR/shacl/#shapes-graph). These are used to validate an
[RDF Graph](https://www.w3.org/TR/rdf11-concepts/#section-rdf-graph) to produce
a validation report.

## Overview

Validation of a RDF Datasets is performed by associating a shapes graph with
each graph (named graph and default graph) in the datasets, then applying the
SHACL validation process to each graph.

This is achieved by associating a "target graph" with each shape. These can be
specified on each shape but can be written to apply to groups of shapes.

Shapes graphs can be reused so that the same set of shapes is used to validate
multiple graphs in the dataset. The shapes to used for a dataset are contained
in a "Shapes Dataset", a collection of shapes graphs and which defines the
shapes to be applied to an RDF datasets with different sets of shapes for
different graphs in the dataset and sharing of shapes between uses on target
graphs.

The declaration
```prefix shx: <http://www.w3.org/ns/shacl-x#>```
is assumed in the definitions.

### Target Graphs

`shx:targetGraph` defines graphs to apply shapes to.
The property can be repeated.

The subject defines scope - it can be a specific shape, a named graph or the
whole dataset (when the subject is the base URI of the shapes dataset).

Certain URIs define a collection of graphs in the target dataset.

| URI | Definition | 
|-----|------------|
| `shx:targetGraph`        | Property identifying a data graph to target  |
| `shx:targetGraphExclude` | Property identifying a data graph to exclude as a target   |

`shx:targetGraphExclude` defines graphs to be excluded from the target graphs.
This target condition is applied after calculating the included target graphs.

## Special Graph Names

| URI | Graphs     | 
|-----|------------|
| `shx:all`     | All the graphs (named and default) of the data dataset.  |
| `shx:named`   | All the named graphs of the data dataset.  |
| `shx:default` | The default graph of the data dataset.  |
| `shx:union`   | The union graph formed from all named graphs of the data dataset.  |

Validating with a target of `shx:union` is performed on the union of the named
graph so the triples used to fulfil a shape may be in different graph. For
example, if a shape defines a number of required properties ("`sh:minCount 1`"),
these may be in different named graphs, but if present in the union graph,
validating suceeds.

## Grouping Shapes

The graphs of the shapes dataset can be used to group shapes. If a
`shx:targetGraph` triple has a graph name as subject, then the target rule is
applied as the default rule for all shapes in that shapes graph. The shape graph
graph name does not need to correspond to any graph in the data to be validated.

```
GRAPH <#g> {
   <#g> shx:targetGraph data:graphName .
   ... any shapes in graph <#g> will have the targetGraph rule above ...
}
```

This does not need to go the named graph itself - it can appear in the default
graph of the shapes dataset.

```
<#g> shx:targetGraph data:graphName .
GRAPH <#g> {
   ... any shapes in graph <#g> will have the targetGraph rule above ...
}
```
Similarly, `shx:targetGraphExclude` also applies to the whole graph.

### Sharing common shapes patterns

The property `shx:include` can be used to include triples from another named
graph of the shapes dataset. A typical use is to include shapes from another
graph which does not have a `shx:targetGraph`, using it like a includes library.

If the subject is the URI of the shapes dataset, the inclusion is into 
declaration applies to the whole dataet (all shapes graphs), otherwise it is to
the named graph of the subject of the triple.

It is similar to `owl:imports` except the inclusion is named by the subject and
only applies to including graphs within the shapes dataset.

`shx:include` is applied transitively.

## Validation Reports

Each validation result in a validation report should have a `sh:resultGraph`
triple whose object is the graph in which the validation report occurs. It is
shape's `shx:targetGraph` if that is given, the graph name matching
`shx:targetGraphPattern` or the specific named graph of the data dataset in
the case of `shx:named` or `shx:all`.

| URI | Definition | 
|-----|------------|
| `shx:targetGraphPattern` | Data graph cause the validation result |

## Examples

Examples are written in [TriG](https://www.w3.org/TR/trig/).

Assumed prefixes:
```
PREFIX rdf:     <http://www.w3.org/1999/02/22-rdf-syntax-ns#> 
PREFIX rdfs:    <http://www.w3.org/2000/01/rdf-schema#>
PREFIX xsd:     <http://www.w3.org/2001/XMLSchema#>

PREFIX sh:      <http://www.w3.org/ns/shacl#>
PREFIX shx:     <http://www.w3.org/ns/shacl-x#>

PREFIX data:    <http:/example/data/>
PREFIX s:       <http:/example/shapes/>
PREFIX ns       <http:/example/ns#>
```

### Examples

This is in the default graph of the shapes dataset which applies to
`data:namedGraph1`.

```
:nodeShape0
    shx:targetGraph data:namedGraph1 ;
    sh:targetObjectsOf ns:q ;
    sh:datatype xsd:integer ;
    .
```

### Example

All shapes applied to a specific named graph.

```
<> shx:targetGraph data:namedGraph1 .

:nodeShape0
    sh:targetObjectsOf ns:q ;
    sh:datatype xsd:integer ;
    .

:nodeShape1
    sh:targetObjectsOf ns:p ;
    sh:datatype xsd:subject ;
    .
```

### Example

A collection of shapes for one graph.

```
GRAPH <#g1> {
     <#g1> shx:targetGraph data:namedGraph1 .

    :nodeShape0
        sh:targetObjectsOf ns:q ;
        sh:datatype xsd:integer .

    :nodeShape1
        sh:targetObjectsOf ns:q ;
        sh:class ns:class .
}
```

### Example
Apply to all graphs except `data:namedGraph1`.

```
<> shx:targetGraph data:all ;
   shx:targetGraphExclude data:namedGraph1 .
```

## Possible additions

### Graph name patterns

For targets, a regular expression to filter the data dataset graph names based
on URIs:

```
[] shx:targetGraphPattern "foobar" .
```

```
[] shx:targetGraphPattern ("foobar" "i") .
```

Similarly `shx:targetGraphExcludePattern`.

### Dataset-wide constraints

Constraints on graphs and graph names (e.g. URI pattern)

These can be done by escaping to SPARQL and a global constraint but if there are
some common constraints, then including a direct form would be useful.

## Notes:

Missing types:

| URI | Type Definition | 
|-----|-----------------|
| `sh:ShapesGraph`         | The class of Shapes Graphs    |
| `shx:ShapesDataset`      | The class of Shapes Datasets  |


The "Shapes Dataset" could be a single graph with just the `shx:targetGraph` and
`shx:targetGraphExclude` vocabulary applied to every shape.  This is a special
case of the design above - `shx:targetGraph` on shapes and using the default
graph only of the TriG file.
