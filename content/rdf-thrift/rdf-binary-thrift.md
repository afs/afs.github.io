---
layout: doc
title: RDF Binary encoding using Thrift
---

Binary RDF is useful for efficient processing and transfer rather than relying
on text-based formats.  Text based formats are more expensive to write and to
parse.

The text oriented syntaxes
(e.g. [Turtle](http://www.w3.org/TR/turtle/)) provide human readability
while the line-oriented syntaxes
(e.g. [N-Triples](http://www.w3.org/TR/n-triples/) provide reasonable
universal dump and machine exchange formats on the web.  N-triples, despite
being larger than Turtle, is generally reported to be faster to parse.

[Apache Thrift](http://thrift.apache.org/), and Google's 
[Protocol Buffers](https://developers.google.com/protocol-buffers/) are
data formats designed for efficient exchange of data between co-operating
processes. This data exchange may be via disk or network.  The formats are
design for efficient processing and not human readability.

RDF Binary is a uses [Apache Thrift](http://thrift.apache.org/) for a binary
encoding for fast machine encoding and decoding.
[Apache Thrift](http://thrift.apache.org/) provides libraries for reading
and writing the encoding in a wide variety of programming languages. 

## Extended RDF Terms {#rdf-terms}

The datastructures of graph/datasets and
SPARQL result sets (e.g. [in JSON](http://www.w3.org/TR/sparql11-results-json/))
build on [RDF Terms](http://www.w3.org/TR/rdf11-concepts/#dfn-rdf-term).
RDF Thrift defines RDF Terms in Thrift and
adds some forms to give a single set of basic building blocks. 

* RDF Terms -- IRIs, literals and blank nodes
* Prefixed names -- shortened IRIs for compression. See prefixed names in Turtle or SPARQL.  
* Named variables (as in SPARQL)
* `ANY` -- a wildcard often found in RDF APIs for finding triples etc.
* `UNDEF` -- explicit indication that something does not have a value (e.g. for SPARQL results).

A possible future (meta) term is 

* `REPEAT` -- a marker to indicate the same term is to be used as the previous "row" 

In addition, there is a way to declare a prefix and the associated IRI. 

These basic elements are used for
 
* triples and quads for graphs and datasets
* rows of values, for SPARQL result sets

This makes for reuse of code to process these different datastructures and simplicity,
efficiency and easy of implementation with the assumption that each datastructure
built on top is careful about erroneous use of these elements.  

### Prefix Declarations {#prefix-decl}

Prefix declarations can be inserted at any point where a triple or quad is expected
and apply from that the end of the declaration in the data stream.

## Graphs and Datasets {#graphs-datasets}

    Content type: application/rdf+thrift
    File extensions: rt, trdf

* Triple (3 RDF terms)
* Quad (4 RDF terms)
* Prefix declaration.

A graph is a stream comprised only of prefix declarations and triples.  

A dataset is a stream comprised only of prefix declarations, triples and quads.
(A triple is in the default graph; quads go into named graphs.)

This format is like N-Triples and N-Quads, except encoded in binary, 
and with prefixed names as a way to write IRIs.  This means the prefix declarations
travel with the triples and quads as they do with Turtle and TriG. 

Prefix declarations can be inserted at any point where a triple or quad is expected
and apply from that the end of the declaration in the data stream.

## SPARQL Result Set {#sparql-result-sets}

    Content type: application/sparql-results+thrift
    File extensions: srt

* Header row: a list of variables.
* Data row: a list of RDF terms to match the header.
* Prefix declaration.

A SPARQL Result Set has one row of variables (the header row) and
zero or more rows of data rows. The header row is mandatory.

All data rows are the same length as the header row.

Prefix declarations can be inserted at any point where a row is expected
and apply from that the end of the declaration in the data stream.

A data row uses `UNDEF` to indicate that a variable not set in the result set for this

In addition, a data row may use the term `REPEAT` to indicate that the same
RDF term from the previous row is used again.  `REPEAT` is illegal in the
first data row.

(`REPEAT` subject to deployment testing that it does provide useful data reduction.)

## Details of Thrift Encoding {#thrift-encoding}

This section details the encoding in Thrift.

### Encoding values {#encoding-numerical-values}

Use of encoding literals by value is optional.

Some literals can be encoded using their values, rather than their lexical
form and datatype.  This can lead to a reduction is space.  It can result
in the changes to the term encoded.

The exact lexcial form is not retained, so `"001"^^xsd:integer`,
`"+1"^^xsd:integer` and `"1"^^xsd:integer` are all encoded as the value `1`
and when read, will result in the same RDF term (`"1"^^xsd:integer`
is the XSD canonical form).

Some derived dataypes are lost - `xsd:long`, `xsd:int`, `xsd:short`,
`xsd:byte` are encoded as their integer value and will become `xsd:integer`
when read back in.

| Input Datatype        | Value space | Outcome Datatype        |
|:----------------------|:------------|:------------------------|
|<tt>xsd:integer</tt>   | Integer     | <tt>xsd:integer</tt>    |
|<tt>xsd:long</tt>      | Integer     | <tt>xsd:integer</tt>    |
|<tt>xsd:int</tt>       | Integer     | <tt>xsd:integer</tt>    |
|<tt>xsd:short</tt>     | Integer     | <tt>xsd:integer</tt>    |
|<tt>xsd:byte</tt>      | Integer     | <tt>xsd:integer</tt>    |
|<tt>xsd:decimal</tt>   | Decimal     | <tt>xsd:decimal</tt>    |
|<tt>xsd:double</tt>    | Double      | <tt>xsd:double</tt>     |

Whether this matters, depends on the data.  For use as a failthful, general
purpose database dump format, value encoding should not be used.

### Prefixes {#encoding-prefixes}

The use of prefixes can reduce the size of the data because it replaces
common character sequences with a smaller string.

There are no detailed syntax rules for prefixes, unlike 
[Turtle](http://www.w3.org/TR/turtle/#grammar-production-PrefixedName)
and [SPARQL](http://www.w3.org/TR/sparql11-query/#rPrefixedname) where, for
example, the local part can not include a `#` (it must be `\#`) and the
local part can't start with a `.`.

In RDf Thrift, the prefix part is any string, the local part is any string.  The
reconsistuted URI is the concatenation of the URI for the prefix and the
local part.

There are no escape sequences in either part.  Neither `%` nor `\` are special.

## Thrift encoding of RDF Terms {#encoding-terms}

RDF Thrift uses the Thrift compact protocol.

    struct RDF_IRI {
    1: required string iri
    }
    
    struct RDF_PrefixName {
    1: required string prefix ;
    2: required string localName ;
    }
    
    struct RDF_BNode {
    1: required string label
    }
    
    struct RDF_Literal {
    1: required string  lex ;
    2: optional string  langtag ;
    3: optional string  datatype ;
    4: optional RDF_PrefixName dtPrefix ;
    }
    
    struct RDF_Decimal {
    1: required i64  value ;
    2: required i32  scale ;
    }
    
    struct RDF_VAR {
    1: required string name ;
    }
    
    struct RDF_ANY { }
    
    struct RDF_UNDEF { }
    
    struct RDF_REPEAT { }
    
    union RDF_Term {
    1: RDF_IRI          iri
    2: RDF_BNode        bnode
    3: RDF_Literal      literal
    4: RDF_PrefixName   prefixName 
    5: RDF_VAR          variable
    6: RDF_ANY          any
    7: RDF_UNDEF        undefined
    8: RDF_REPEAT       repeat
    9: RDF_Triple       tripleTerm  # RDF-star
    
    # Value forms of literals.
    10: i64             valInteger
    11: double          valDouble
    12: RDF_Decimal     valDecimal
    }

## Thrift encoding of Triples, Quads and rows. {#encoding-tuples}

    struct RDF_Triple {
    1: required RDF_Term S
    2: required RDF_Term P
    3: required RDF_Term O
    }
    
    struct RDF_Quad {
    1: required RDF_Term S
    2: required RDF_Term P
    3: required RDF_Term O
    4: optional RDF_Term G
    }
    
    struct RDF_PrefixDecl {
    1: required string prefix ;
    2: required string uri ;
    }

## Thrift encoding of RDF Graphs and RDF Datasets {#encoding-graphs-datasets}

    union RDF_StreamRow {
    1: RDF_PrefixDecl   prefixDecl
    2: RDF_Triple       triple
    3: RDF_Quad         quad
    }

RDF Graphs are encoded as a stream of `RDF_Triple` and `RDF_PrefixDecl`.

RDF Datasets are encoded as a stream of `RDF_Triple`, `RDF-Quad` and `RDF_PrefixDecl`.

## Thrift encoding of SPARQL Result Sets {#encoding-result-sets}

A SPARQL Result Set is encoded as a list of variables (the header), then
a stream of rows (the results).

    struct RDF_VarTuple {
    1: list<RDF_VAR> vars
    }
    
    struct RDF_DataTuple {
    1: list<RDF_Term> row
    }
