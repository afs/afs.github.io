---
layout: doc
title: RDF and IRI Syntax
---
<div class="docinfo">
     <p>Andy Seaborne</p>
     <p>February, 2010</p>
</div>

RDF uses IRIs to refer to resources.

This page pulls together the syntax requirements for IRIs into one place. There
are several documents, mostly RFCs, that define IRIs and specific IRI
schemes. Getting IRIs right means data is more likely to be readable by
other systems when the data is published.

Just because an IRI passes all the syntax rules, it does not make it a good choice.

URIs are defined by [RFC 3986](https://tools.ietf.org/html/rfc3986]) and 
IRIs in the companion [RFC 3987](https://tools.ietf.org/html/rfc3987]) which
gives the modifications necessary for the wider range of
Unicode characters in URIs.

URI schemes can add constraints on the URI syntax; for example, [RFC
7230](https://tools.ietf.org/html/rfc7230) defines the http and https shemes.

This article introduces the terminology 'RDF Reference' to put all the
implications into one definition.


## TOC {#toc}

- [IRIs](#iris)
  - [Definition of URI syntax](#defn-uri-syntax)
  - [Relative References](#relative-references)
  - [Example](#example)
  - [Notes about IRIs/URIs](#notes-iris)
  - [Normalization](#normalization)
- [HTTP](#http)
- [URN](#urn)
- [UUIDs](#uuid)
- ["file" URI scheme](#file)
- [RDF References](#rdf-references)
- [Gotchas](#gotchas)
- [Links](#links)

## IRIs

This article will use URI and IRI interchangeably. 
"[IRIs are a generalization of URIs that permits a wider range of
Unicode characters.](https://www.w3.org/TR/rdf11-concepts/#note-iris)"

The [RDF Concepts document](https://www.w3.org/TR/rdf11-concepts/#section-IRIs) says:

> "IRIs in the RDF abstract syntax MUST be absolute, and
MAY contain a fragment identifier."

> "Relative IRIs must be resolved against a base IRI to make them absolute."

As of [RFC 3986](https://tools.ietf.org/html/rfc3986), relative IRIs are called
"relative references".

### Definition of URI syntax {#defn-uri-syntax}

In [RFC3986 section-4.1](https://tools.ietf.org/html/rfc3986#section-4.1)
defines "URI".

A "URI" is URI-reference after it has been resolved.

The relevant part of the grammar in 
[RFC3986 appendix-A](https://tools.ietf.org/html/rfc3986#appendix-A)
is:

```
   absolute-URI  = scheme ":" hier-part [ "?" query ]

   URI           = scheme ":" hier-part [ "?" query ] [ "#" fragment ]

   hier-part     = "//" authority path-abempty
                 / path-absolute
                 / path-rootless
                 / path-empty
```

We've already got to one important point - an _absolute URI_. An absolute URI has
a URI scheme, and does not have a fragment.  

An absolute URI with a fragment is just "URI".

What we want is the full URI rule and also require it uses the "// authority"
rule if a scheme involves that component.

Specifc URI schemes can add additional requirments. 
HTTP ([RFC 7230](https://tools.ietf.org/html/rfc7230))
requires an http URI to have the part `"//" authority`; 
the urn scheme does not have an authority.

URN ([RFC 8141](https://tools.ietf.org/html/rfc8141)) does not have an authority
part - it uses the `path-rootless` production. It does additionally require the
path to have two colons, the NID part must be at
least two characters, and the NSS part at least one character.

### Relative References

RDF syntax may use 
[relative references](https://www.w3.org/TR/rdf11-concepts/#dfn-relative-iri).
The process of parsing of a document means that any
relative reference is converted to an URI to ensure it identifies the same
resource everywhere. This is called resolving against a base URI - 
[there is always a base](https://tools.ietf.org/html/rfc3986#section-5).

Relative references are short cuts, for the full URI after resolving
against the base.

By the time they get to RDF abstrat syntax (the datastructure), relative
references should have been converted.

### Example

By the definitions in RFC 3986:

```
    http:abcd
    http:/abcd
```

are absolute URIs - it has a URI scheme, it does not have a fragment.

But it is not an "[http-URI](#http)" which is defined in
[RFC7230 sec 2.7.1](https://tools.ietf.org/html/rfc7230#section-2.7.1)
because http-URI defines additional syntax requirements. An http-URI requires
the `"//" authority path-abempty` choice in `hier-part`.

With [non-strict resolution](https://tools.ietf.org/html/rfc3986#section-5.2.2),
and an valid HTTP base URI, the examples won't appear in RDF abstract syntax.

### Notes about IRIs/URIs {#notes-iris}

* Space (U+0200) is not legal in an IRI.
* `@`, `~`, `(`, `)` and `:` are legal in the path, query and fragment components.
* `{` and `}` are not legal in IRIs.
* `[` and `]` are only legal as IPv6 address delimiters.
* Encoding with "%hex" is not an escape mechanism. `%20` puts three characters into
the IRI: `%`, `2`, `0`.

### Normalization

Normalization of the syntax 
([RFC 3998, section 6.2.2](https://tools.ietf.org/html/rfc3986#section-6.2.2)
gives some simple rules to make it easier to compare URIs as strings.

By the URI syntax the two characters after `%` 
must be legal hex digits (`%ST` is a syntax error).
Normalization prefers the letters "A-F" to be uppercase.

## HTTP {#http}

[RFC7230 sec 2.7.1](https://tools.ietf.org/html/rfc7230#section-2.7.1)

```
  http-URI = "http:" "//" authority path-abempty [ "?" query ] [ "#" fragment ]
```

so the http URI scheme adds a requirement that there must be a "//" and
authority (host and port), followed by an absolute path (starts with "/") or is
absent (empty string, no "/").

## URN {#urn}

The
[syntax of the urn URI scheme](https://tools.ietf.org/html/rfc8141#section-2)

```
      assigned-name = "urn" ":" NID ":" NSS
      NID           = (alphanum) 0*30(ldh) (alphanum)
      ldh           = alphanum / "-"
      NSS           = pchar *(pchar / "/")
```

The older [RFC 2141](https://tools.ietf.org/html/rfc2141) allowed "X-..." as NID.

While a URN allows `r-component`, `q-component` and `f-component`, the latter
being a URI fragment, usually it is just the `assigned-name` form used for
resource identification.

The urn scheme only applies to ASCII, not the addition characters of IRIs.

An NID must be at least 2 characters, and the first and last must be
alphanumeric.

A NSS must be at least one character.

It is not uncommon to see `urn:x:...` in test data - unfortunately, that isn't a
legal URI in the urn scheme because the "x" is the NID part and is too short.

By the general URI syntax, the URI path component is the "NID:NSS" part of the URN.

## UUIDs {#uuid}

The correct way to use UUIDs 
([RFC 4122](https://tools.ietf.org/html/rfc4122#section-3))
is `urn:uuid:0e5f5ff6-6c80-4786-84b9-4c121bb3ae9e`.
Hex digits should be lower case.

There was a [proposal](https://tools.ietf.org/html/draft-kindel-uuid-uri-00) for
a "uuid" scheme, it is is sometimes seen but it was only ever a draft. 

## "file" URI scheme {#file}

The file URI scheme had for a long time been only loosely defined in
[RFC 1738 section 3.10](https://tools.ietf.org/html/rfc1738#section-3.10).
Common usage was beyond the definition; character set issues were unclear.

[RFC 8089](https://tools.ietf.org/html/rfc8089) is a formal defintion compatible
with the URI syntax for RFC 3986.  It includes common usage such as relative
filenames (relative to their point of use), for example
`file:directory/file.txt`, can be used.

While for RDF the file schema is of limited use, the file URI scheme is useful
when working with local files, for example using RDF for configuration files on the
local machine. Such URIs will be of the form `file:///` (that is, 3 '/') using
the file scheme convention that "localhost" can be dropped.

## RDF References {#rdf-references}

It woudl be useful to pull all these considerations together into a distinct
piece of terminolgy 

"RDF References":

* It is a URI

That means after resolving against a base.

It has a scheme name. It may have query and fragment parts. There is always a
"path" even if it is the empty string.

* It SHOULD NOT have a "userinfo@", the user-password part of authority.  
  This is deprecated in 
  [RFC 3986 section 3.2.1](https://tools.ietf.org/html/rfc3986#section-3.2.1).

* It follows the additional restrictions of the URI scheme.

This can be tricky for the parser to check if it does not know the scheme.

* The scheme-specific rules for http, https and urn schemes are required:
   * If 'http:' it follows the HTTP scheme rule: `http-URI`
   * If 'urn:', it matches the requirment for "urn:2+chars:1+char"

* Hex in %-encoding SHOULD be uppercase.

## Gotchas

In Turtle related synatxes, there are two places where "partial URIs" are used.

```
PREFIX u: <urn:uuid:>

u:66d5b9e2-5abe-49be-bfc9-1ed0d997e07f
```
but `urn:uuid:` is not a legal URN.

```
BASE <urn:>

## Resolves to urn:uuid:66d5...
<uuid:66d5b9e2-5abe-49be-bfc9-1ed0d997e07f>
```

but `urn:` is not a legal URNs.

## Links

Links to relevant documents:

|           | Defined by: |
|-----------|-------------| 
| URI       | [RFC 3986](https://tools.ietf.org/html/rfc3986) |
| IRI       | [RFC 3987](https://tools.ietf.org/html/rfc3987) |
| http:     | [RFC 7230](https://tools.ietf.org/html/rfc7230) |
| urn:      | [RFC 8141](https://tools.ietf.org/html/rfc8141) |
| urn:uuid: | [RFC 4122](https://tools.ietf.org/html/rfc4122) |
| did:      | [W3C DID](https://www.w3.org/TR/did-core/)      |
| file:     | [RFC8089](https://tools.ietf.org/html/rfc8089)  |

<br/>Registries:

* [IANA URI schemes](https://www.iana.org/assignments/uri-schemes/uri-schemes.xhtml)
* [IANA URN
namespaces](https://www.iana.org/assignments/urn-namespaces/urn-namespaces.xhtml)
