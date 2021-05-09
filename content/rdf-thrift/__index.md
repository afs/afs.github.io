---
layout: doc
title: RDF Binary using Apache Thrift
slug: index
---

RDF Binary using Apache Thrift ("RDF Thrift") is a binary format
for RDF and RDF-related data.

The W3C standard RDF syntaxes are text or XML based.  These incur costs in
parsing; the most human-readable formats also incur high costs to write, and
have limited scalability due to the need to analyse the data for pretty
printing rather than simply stream to output.

[N-Triples](http://www.w3.org/TR/n-triples/) or
[N-Quads](http://www.w3.org/TR/n-quads/) are often used for datastore dump
formats and for publishing large datasets because they have been found to
be the best formats to read and write for this usages.  Yet these are still
text formats.

SPARQL result sets have a number of related syntaxes, based on 
[using JSON](http://www.w3.org/TR/sparql11-results-json/), 
[using XML](http://www.w3.org/TR/rdf-sparql-XMLres/), or
[using TSV or CSV](http://www.w3.org/TR/sparql11-results-csv-tsv/).
Again, these are text-based formats.

Binary formats are faster to process - they do not incur the parsing costs of text-base
formats.  "RDF Thrift" defines basic encoding for RDF terms then builds
data formats for RDF graphs, RDF datasets and for SPARQL result sets.
This gives a basis for high-performance linked data systems.

[Apache Thrift](http://thrift.apache.org/) provides a efficient, 
wide-used binary encoding layer with a large number of language bindings.
(Thrift also provides a cross-language, service interaction model on top of this
encoding layer - this is not used here.)

* Format description: [RDF Binary encoding using Thrift](rdf-binary-thrift.html).

* Known implementations: 
  * Apache Jena: [Jena API to RDF Trift](https://jena.apache.org/documentation/io/rdf-binary.html).

Document license: [Apache License](LICENSE.txt) .
