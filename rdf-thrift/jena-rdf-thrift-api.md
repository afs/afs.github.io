---
layout: doc
title: Apache Jena implementation of RDF Binary
---

RDF Thrift has been incorporated in to apache Jena (as of 2014-08-31).

A version of Jena built after that date is required (Jena 2.12.1-SNAPSHOT or later).

## Comand line 

The `riot` command has an output format argument:

    riot --out=TRDF data.nt > data.trdf

## Graphs and Datasets  

Use `RDFDataMgr` operations.

Read a graph:

    Model m = RDFDataMgr.loadModel("data.trdf") ;

Write a model:

    RDFDataMgr.write(System.out, m, Lang.RDFTHRIFT) ;
    
and other `RDFDataMgr` operations. 

Reading using `RDFDataMgr` is preferred.  `model.read` operations may not detect
the file syntax.

## SPARQL Result Sets

    ResultSet resultSet =  ResultSetMgr.read(inputStream, ResultSetLang.SPARQLResultSetThrift) ;

    ResultSetMgr.write(outputStream, resultSet,  ResultSetLang.SPARQLResultSetThrift) ;

## Experimentation

If you want to access the code for the Thrift encoding, see the classes: 

* `BinRDF` -- the read and write operations.
* `ThriftConvert` -- conversion to and from Thrift datastructures.
* `TRDF` -- support code for handling Thrift I/O (e.g. `TProtocol`).
