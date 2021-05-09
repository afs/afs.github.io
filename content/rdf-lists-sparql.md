---
layout: doc
title: Updating RDF Lists with SPARQL
---
<div class="docinfo">
     <p>Andy Seaborne</p>
</div>

<i>Migrated from the original (2011):
[Updating RDF Lists with SPARQL](http://seaborne.blogspot.com/2011/03/updating-rdf-lists-with-sparql.html).</i>

Something the [SPARQL 1.1 Working Group](http://www.w3.org/2009/sparql/wiki/Main_Page)
discussed was updates to RDF lists.

RDF lists are hard to deal with because they are not [first class
objects](http://en.wikipedia.org/wiki/First-class_object) in the RDF data
model. Instead they are "encoded" in triples. The encoding using a
[cons cell](http://en.wikipedia.org/wiki/Cons)-like structure whereby each
element of the list is a blank node (not necessary a blank node but it nearly
always is). It's a one-way link list.

RDF lists are correctly called "[RDF collections](http://www.w3.org/TR/rdf-primer/#collections)" 
but as it's the list-nature (elements in order) that matters, I'll call them lists in this note.

Turtle and SPARQL has syntax for lists, but it's only surface syntax, and there
are really triples in the RDF graph:

    PREFIX : <http://example/>
    :x :p (1 2 3) .

is the RDF:

    :x    :p         _:b0 .
    _:b0  rdf:first  1 .
    _:b0  rdf:rest   _:b1 .
    _:b1  rdf:first  2 .
    _:b1  rdf:rest   _:b2 .
    _:b2  rdf:first  3 .
    _:b2  rdf:rest   rdf:nil

RDF toolkits help by presenting lists as progamming language lists.  This also
helps in keeping the lists well formed.  In all those triples, there is one
`rdf:rest` and one `rdf:first` per list element - but it's legal RDF to have
several uses of the properties, or none, on one subject.

As an addition quirk, the empty list isn't any RDF triples, so looking for lists isn't just looking for <tt>rdf:rest</tt> properties. </p>

    PREFIX : <http://example/>
    :x :p () .

is the RDF:

    :x :p rdf:nil .

### Lists, Property Paths and Update

[SPARQL 1.1 Query](http://www.w3.org/TR/sparql11-query/) adds 
[property paths](http://www.w3.org/TR/sparql11-query/#propertypaths), which make working
with lists a bit easier, but it's not perfect.  List elements do not necessarily
come out in order.

    { :list rdf:rest*/rdf:first ?element }


But what about [SPARQL 1.1 Update](http://www.w3.org/TR/sparql11-update/)?  
How can we work with RDF lists?  Here are some scripts for list operations.  By
using property paths they work on arbitrary length lists.


* [Add an element to the start of a list](#add-first)
* [Add an element to the end of a list](#add-last)
* [Delete the element at the start of a list](#del-first)
* [Delete an element to the end of a list](#del-last)
* [Delete the whole list](#del-all-1) (version 1 - common case)
* [Delete the whole list](#del-all-2) (version 2 - more general case)

All the scripts are self-contained - they include tests data.

They are examples - they aren't necessarily fully general, for example, if lists
are badly formed or the property `:p` is also used to relate the subject to
things that aren't lists.  The last example shows a way to address that by
finding and marking relavent points in the graph, doing some work and going back
and tidying up.  The graph updated is also being used as a scratch pad.

## <a name="add-first"></a>Add an element to the start of a list

    PREFIX :    &lt;http://example/> 
    PREFIX rdf: &lt;http://www.w3.org/1999/02/22-rdf-syntax-ns#> 
    
    INSERT DATA {
      :x0 :p () .
      :x1 :p (1) .
      :x2 :p (1 2) .
      :x3 :p (1 2 3) .
    } ;
    
    DELETE { ?x :p ?list }
    INSERT { ?x :p [ rdf:first 0 ; 
                     rdf:rest ?list ]
    } WHERE {
      ?x :p ?list .
    }

This one is relatively easy. Find the list start <tt>?x :p ?list</tt>, which works whether the list is zero length or already has elements,  delete the old triple that connected to the start of the list, put in a new cons cell (the <tt>[...]</tt>) at the start, and link to it.

### <a name="add-last"></a>Add an element to the end of a list

    PREFIX :    &lt;http://example/> 
    PREFIX rdf: &lt;http://www.w3.org/1999/02/22-rdf-syntax-ns#> 
    
    INSERT DATA {
      :x0 :p () .
      :x1 :p (1) .
      :x2 :p (1 2) .
      :x3 :p (1 2 3) .
    } ;
    
    # The order here is important.
    # Must do list >= 1 first.
    
    # List of length >= 1
    DELETE { ?elt rdf:rest rdf:nil }
    INSERT { ?elt rdf:rest [ rdf:first 98 ; rdf:rest rdf:nil ] }
    WHERE
    {
      ?x :p ?list .
      # List of length >= 1
      ?list rdf:rest+ ?elt .
      ?elt rdf:rest rdf:nil .
      # ?elt is last cons cell
    } ;
    
    # List of length = 0
    DELETE { ?x :p rdf:nil . }
    INSERT { ?x :p [ rdf:first 99 ; rdf:rest rdf:nil ] }
    WHERE
    {
       ?x :p rdf:nil .
    }

This is a bit harder - there are two cases, lists of length 0 and lists of
length one or more. The element before the insertion point needs changing and
that can be a cons cell (list length >= 1) or the empty list (the triple
pointing to it).

Do the lists of length one or more first, otherwise the adding to a list of
length zero will be caught again by the adding to a list of length one.

For a list of length 1 or more: find the last element.  The <tt>WHERE</tt> finds <tt>?elt</tt> by finding all elements of the list <tt>rdf:rest+</tt>, and checking it's the last element by looking for <tt>?elt rdf:rest rdf:nil</tt>.

Then delete the <tt>rdf:rest</tt>, and insert the new cons cell 
<tt>[ rdf:first 98 ; rdf:rest rdf:nil ]</tt>.

For a list of length 0, the style is the same but the finding the triple to
delete-insert to attach the cons cell is different.

### <a name="del-first"></a>Delete the element at the start of a list

    PREFIX :      &lt;http://example/> 
    PREFIX rdf:   &lt;http://www.w3.org/1999/02/22-rdf-syntax-ns#> 
    
    INSERT DATA {
      :x3 :p (1 2 3) .
      :x2 :p (1 2) .
      :x1 :p (1) .
      :x0 :p () .
    } ;
    
    DELETE { 
       ?x :p ?list .
       ?list rdf:first ?first ;
             rdf:rest  ?rest }
    INSERT { ?x :p ?rest }
    WHERE
    {
      ?x :p ?list .
      ?list rdf:first ?first ;
            rdf:rest ?rest .
    }

This can be done in one step - we are not interested in lists of length 0
because they have no element to delete.  So find the pattern at the start of the
list, delete it (note the <tt>WHERE</tt> pattern and <tt>DELETE</tt> template
are the same), and insert the new triple that links the list directly to the
previous <tt>rdf:rest</tt>. </p>

### <a name="del-last"></a>Delete the element at the end of a list

    PREFIX :     &lt;http://example/> 
    PREFIX rdf:  &lt;http://www.w3.org/1999/02/22-rdf-syntax-ns#> 
    
    INSERT DATA {
      :x3 :p (1 2 3) .
      :x2 :p (1 2) .
      :x1 :p (1) .
      :x0 :p () .
    } ;
    
    # List of length 1
    # Do before other lists.
    
    DELETE { ?x :p ?elt .
             ?elt  rdf:first ?v .
             ?elt  rdf:rest  rdf:nil .
           }
    INSERT { ?x :p rdf:nil . }
    WHERE
    {
      ?x :p ?elt .
      ?elt rdf:first ?v ;
           rdf:rest rdf:nil .
    } ;
    
    # List of length >= 2
    DELETE { ?elt1 rdf:rest ?elt .
             ?elt  rdf:first ?v .
             ?elt  rdf:rest  rdf:nil .
           }
    INSERT { ?elt1 rdf:rest rdf:nil }
    WHERE
    {
      ?x :p ?list .
      ?list rdf:rest* ?elt1 .
    
      # Second to end.
      ?elt1 rdf:rest ?elt .
      # End.
      ?elt rdf:first ?v ;
       rdf:rest rdf:nil .
}

The cases to consider are lists of exactly one and lists of two or more
elements.  It's the treatment of the element before the element we're deleteing
that is different.

The style is the same though - find the place before the deleting, and the
delete that cons cell.

For the list of length 2 or more, <tt>rdf:rest*</tt> is used which, is all
elements including the <tt>?list</tt> case of zero steps - then the structure
beyond that is tested for being the end.  There are 2 <tt>rdf:rest</tt> uses in
the test for the end, hence list of length 2 or more.

### <a name="del-all-1"></a>Delete the whole list (common case)

    PREFIX rdf:  &lt;http://www.w3.org/1999/02/22-rdf-syntax-ns#> 
    PREFIX :     &lt;http://example/> 
    
    INSERT DATA {
        :x0 :p () .
        :x0 :q "abc" .
        
        :x1 :p (1) .
        :x1 :q "def" .
        
        :x2 :p (1 2) .
        :x2 :q "ghi" .
    } ;
    
    # Delete the cons cells.
    DELETE
        { ?z rdf:first ?head ; rdf:rest ?tail . }
    WHERE { 
          [] :p ?list .
          ?list rdf:rest* ?z .
          ?z rdf:first ?head ;
             rdf:rest ?tail .
          } ;
    
    # Delete the triples that connect the lists.
    DELETE WHERE { ?x :p ?z . }

This version is not fully general because it assume that <tt>:p</tt> is a link
to the list and not also to any other RDF terms (non-lists) which we would want
to keep.

The first <tt>DELETE</tt> finds and removes all cons cells.  The second
<tt>DELETE</tt> removes the triple with <tt>:p</tt> connecting the list to the
subject.

### <a name="del-all-2"></a>Delete the whole list (general case)

    PREFIX rdf:  &lt;http://www.w3.org/1999/02/22-rdf-syntax-ns#> 
    PREFIX :     &lt;http://example/> 
    
    INSERT DATA {
        :x0 :p () .
        :x0 :p "String 0" .
        :x0 :p [] .
        
        :x1 :p (1) .
        :x1 :p "String 1" .
        :x1 :p [] .
        
        :x2 :p (1 2) .
        :x2 :p "String 2" .
        :x2 :p [] .
        
        # A list not connected.
        (1 2) .
        
        # Not legal RDF.
        # () .
        
    } ;
    
    INSERT { ?list :deleteMe true . }
    WHERE {
       ?x :p ?list . 
       FILTER (?list = rdf:nil || EXISTS{?list rdf:rest ?z} )
    } ;
    
    # Delete the cons cells.
    DELETE
        { ?z rdf:first ?head ; rdf:rest ?tail . }
    WHERE { 
          [] :p ?list .
          ?list rdf:rest* ?z .
          ?z rdf:first ?head ;
             rdf:rest ?tail .
          } ;
    
    # Delete the marked nodes
    DELETE 
    WHERE { ?x :p ?z . 
            ?z :deleteMe true . 
    } ;
    
    ## ------
    ## Unconnected lists.
    
    DELETE
        { ?z rdf:first ?head ; rdf:rest ?tail . }
    WHERE { 
          ?list rdf:rest ?z2 .
          FILTER NOT EXISTS { ?s ?p ?list }
          ?list rdf:rest* ?z .
          ?z rdf:first ?head ;
             rdf:rest ?tail .
          } 
    
Deep breath.

This one is quite long.

The first step is to find and mark all the triples from a subject to a list via
<tt>:p</tt>. We will need to delete at the end of the process but the property
might also be used for non-lists and after the middle <tt>DELETE</tt> step all
evidence of the lists is lost. The test:

    FILTER (?list = rdf:nil || EXISTS{?list rdf:rest ?z} )

catches both zero length lists and lists with elements.

Second step: delete all list elements, any subjects with properties <tt>rdf:first</tt> and <tt>rdf:rest</tt>.

Third step: remove the connecting triples and the markers.

Finally, we delete any lists where the start isn't connected to anything, which is
the

    FILTER NOT EXISTS { ?s ?p ?list }

test.

### License and Copyright

This page and the SPARQL 1.1 Update scripts are (c) 
<a href="">Epimorphics Ltd</a> and licensed under a 
<a href="http://creativecommons.org/licenses/by/3.0">Creative Commons Attribution 3.0
License</a>.

