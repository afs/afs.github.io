---
layout: doc
title: SPARQL Grouping and Aggregation with no matches
---
<div class="docinfo">
     <p>Andy Seaborne</p>
     <p>March 2020</p>
</div>

Grouping and Aggregation is the combination of `GROUP BY` and one of the
aggregation functions such as `COUNT` or `MAX`.  When there are no matches to
the `WHERE` clause, the spec and one fo the tests don't agree. In the case of aggregation
and no `GROUP BY`, the spec needs a fix to get the expected answer.

* [Description](#description)
* [Grouping and Aggregation, no matching rows](#group-agg-no-rows)
* [Aggregation, no grouping, no matching rows](#agg-no-group-no-rows)
* [Spec text](#spec-text)
* [Execution when grouping has no rows](#exec-no-rows)
* [The "no group" case](#no-group)
* [Possible Fix, "no group" case](#no-group-fix)

## Description {#description}

Informally, what happens in groupign and aggregation is that the matches from
the `WHERE` clause are paritioned by calculating the group key from the `GROUP
BY` expression for each row , with each row going into a collection of rows with
the same group key. We end up with a dictionary mapping group key to a
collection of those rows with the same group key. Each grouping of rows has at
least one row in it because create dictionary entry a row with that group key
must have been encountered.

When there is aggregation, the aggregation function (`COUNT`, `MAX`, etc) is
called on each dictionary entry with the collection of rows as argument.

If there are no rows as a result of the `WHERE` clause, there are no pairs of
group key row collection.

There is a special case of aggregation when there is no `GROUP BY` but there is
an aggregate function. All the rows are collected together with a special group
key of `1`. The dictionary has one entry.

There is has a problem in the specfication here - it puts in a special `GROUP BY`
expression but when there are no rows from the `WHERE` clause, it does not get
called. [A possible solution](#no-group) is given below.

From this we can see that:

* When group+aggregate, there are the same number of rows as the grouping alone.
* If there is aggrgation and no group, there is one row in result.

## Grouping and Aggregation, no matching rows {#group-agg-no-rows}

`GROUP BY`, and `GROUP BY` with aggregation have the same number of rows. 

There is a disprepency between the SPARQL tests and the specification.
The test [aggregates-agg-empty-group](https://www.w3.org/2009/sparql/docs/tests/summary.html#aggregates-agg-empty-group)
(Also found at
[rdf-tests:sparql11/data-sparql11/aggregates](https://github.com/w3c/rdf-tests/tree/gh-pages/sparql11/data-sparql11/aggregates))
runs on an empty graph.

    PREFIX ex: <http://example.com/>
    SELECT ?x (MAX(?value) AS ?max)
    WHERE {
        ?x ex:p ?value
    } GROUP BY ?x

Beause it is an empty graph, the WHERE clause has zero rows, and the group step
is zero rows.

But the
[results](https://www.w3.org/2009/sparql/docs/tests/data-sparql11/aggregates/agg-empty-group2.srx)
are given as:

    <sparql xmlns="http://www.w3.org/2005/sparql-results#">
      <head>
        <variable name="x"/>
        <variable name="max"/>
      </head>
      <results>
        <result> </result>
      </results>
    </sparql>

which is one row, with no values, not zero rows.

    -----------
    | x | max |
    ===========
    |   |     |
    -----------

(the original test results have no `<variable name="x"/>` - that's
corrected in test manifest with [these results](https://www.w3.org/2009/sparql/docs/tests/data-sparql11/aggregates/agg-empty-group2.srx))

It should be:

    -----------
    | x | max |
    ===========
    -----------

that is, zero rows, because

    SELECT ?x
    WHERE {
        ?x ex:p ?value
    } GROUP BY ?x

has zero rows, the grouping is an empty dictionary (no group keys), and
`(MAX(?value) AS ?max)` is not called at all.

## Aggregation, no grouping, no matching rows {#agg-no-group-no-rows}

Compare with the case where there is no `GROUP BY`

    SELECT  (MAX(?value) AS ?max)
    WHERE {
        ?x ex:p ?value
    } 

    -------
    | max |
    =======
    |     |
    -------

where there is supposed to be one row.

Which aggregate is used shouldn't affect the number of rows, and `MAX({})` is
defined to be an evaluation error. The `COUNT` case is clearer:

    SELECT ?x (COUNT(*) AS ?count) 
    WHERE { ?x ex:p ?value }
    GROUP BY ?x

gives

    -------------
    | x | count |
    =============
    -------------

and

    SELECT  (COUNT(*) AS ?count)
    WHERE { ?x ex:p ?value }

gives

    ---------
    | count |
    =========
    |     0 |
    ---------

The difference is that when there is an empty outcome from `GROUP BY`, there is
an empty dictionary and the aggregation function, `MAX` or `COUNT` isn't called
at all. With no `GROUP BY`, theer is a dictionary enry with an empty collection
of rows and the case `Max({}) = error` covers the case of a call where there
zero rows to aggregate.

    SELECT ?x
    WHERE {
        ?x ex:p ?value
    } GROUP BY ?x

and no matches of `?x ex:p ?value` is zero rows.

## Spec text {#spec-text}

These are the relevant parts of the spec and errata:

Section 18.5.1 https://www.w3.org/TR/sparql11-query/#aggregateAlgebra

Group, produces a dictionary the group key `ListEval(exprlist, μ)` to (`→`) rows `{ μ' | ...}`:

    Group(exprlist, Ω) = { ListEval(exprlist, μ) → { μ' | μ' in Ω, ListEval(exprlist, μ) = ListEval(exprlist, μ') } | μ in Ω }

Aggregation, produces a dictionary group key to aggregate value:

    Aggregation(exprlist, func, scalarvals, { key1→Ω1, ..., keym→Ωm } )
       = { (key, F(Ω)) | key → Ω in { key1→Ω1, ..., keym→Ωm } }

Then `AggregateJoin` puts the results into a query solution.

During evaluation (after [errata-query-11](https://www.w3.org/2013/sparql-errata#errata-query-1)):

    eval(D(G), Aggregation(exprlist, func, scalarvals, Group(exprlist, P)))
        = Aggregation(exprlist, func, scalarvals, eval(D(G), Group(exprlist, P))) 

Calling the aggregate function happens within the `F(Ω)` calling `func`.

## Execution when grouping has no rows {#exec-no-rows}

    Group(exprlist, Ω) = { }

because `μ in Ω` is empty. Then aggregation is on the empty dictionary:

    Aggregation(exprlist, func, scalarvals, { } )
       =  { (key, F(Ω)) | key → Ω in { } }

which is also `{ }` because there is no `(key → Ω) in { }`.

The query result is zero rows.

If `GROUP BY` produces zero rows, aggregation produces zero rows.

## The "no group" case {#no-group}

Use of aggregates when there is no `GROUP` BY is different.

Section 8.2.4.1 https://www.w3.org/TR/sparql11-query/#sparqlGroupAggregate

This puts in a dummy key of `(1)` (a exprlist of one element which is the value 1).

But if there are no rows for the patten matching.

    Group(exprlist, { }) = { }

regardless of the `exprlist` - the dictionary entry using key 1 is not created.

But we expect

    SELECT (COUNT(*) AS ?C)
    WHERE { ?x ex:p ?value }

(no matches to the `WHERE`) to be one row with `?C` being `0`.

If it were `MAX`, then `MAX({})` happens, which evaluates to an error so we have one
row, no columns.

In particular, aggrgeate with no group is _not_ like writing:

    PREFIX ex: <http://example.com/>
    SELECT ?x (COUNT(*) AS ?count)
    WHERE {
        ?x ex:p ?value
    } GROUP BY (1)

which is "no rows" again.

## Possible Fix, "no group" case {#no-group-fix}

Somehow, "no group by, with aggregates" needs to made special.

One way is to make "aggregate, no group" become a distinct, special case of
`Group()` that creates the right dictionary.

8.2.4.1 https://www.w3.org/TR/sparql11-query/#sparqlGroupAggregate<br/>
Step Aggregates

We make the dummy key special marker `'group-all'`, then group all rows into
one dictionary entry, even if the row collection is empty.

This safe because `'group-all'` isn't an exprlist and so can not be written in a
query using `GROUP BY`.

We could use any marker value in place of `'group-all'` as long as it is not an
exprlist. `1`, not `(1)`, would work or `{}`. The group key of `1` is introduced
when `Group` is evaluated as that group keys are RDF terms as before.

The only time `Group('group-all', P)` appears is when there is an aggregation step
and there is no `GROUP BY`.

    If Q contains GROUP BY exprlist
       Let G := Group(exprlist, P)
    Else If Q contains an aggregate in SELECT, HAVING, ORDER BY
       Let G := Group((1), P)
    Else
       ...

becomes:

    If Q contains GROUP BY exprlist
       Let G := Group(exprlist, P)
    Else If Q contains an aggregate in SELECT, HAVING, ORDER BY
       Let G := Group('group-all', P)
    Else
       ...

Then `Group` handles the case `Group('group-all', Ω)` by putting in the group key
of 1 that the spec uses:

    Group('group-all', Ω) = { (1 → Ω) }
    Group(exprlist, Ω) = { ListEval(exprlist, μ) → { μ' | μ' in Ω, ListEval(exprlist, μ) = ListEval(exprlist, μ') } | μ in Ω }

`exprList` is never `'group-all'` so these two evaluation cases don't overlap.
