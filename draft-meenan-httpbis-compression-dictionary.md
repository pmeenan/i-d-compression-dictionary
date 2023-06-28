---
coding: utf-8

title: Compression Dictionary Transport
abbrev: compression-dictionary
category: info

docname: draft-meenan-httpbis-compression-dictionary-latest
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"
number:
date:
consensus: true
v: 0
area: "Applications and Real-Time"
workgroup: "HTTP"
keyword:
 - compression dictionary
 - shared brotli
 - zstandard dictionary
venue:
  group: "HTTP"
  type: "Working Group"
  mail: "ietf-http-wg@w3.org"
  arch: "https://lists.w3.org/Archives/Public/ietf-http-wg/"
  github: "pmeenan/i-d-compression-dictionary"
  latest: "https://pmeenan.github.io/i-d-compression-dictionary/draft-meenan-httpbis-compression-dictionary.html"

author:
  -
    ins: P. Meenan
    name: Patrick Meenan
    org: Google LLC
    email: pmeenan@google.com
  -
    ins: Y. Weiss
    name: Yoav Weiss
    org: Google LLC
    email: yoavweiss@google.com

informative:
  Brotli: RFC7932
  Zstandard: RFC8878
  HTTP: RFC7230
  RFC8941:

--- abstract

This specification defines a mechanism for using designated {{HTTP}} responses
as an external dictionary for future HTTP responses for compression schemes
that support using external dictionaries (e.g. {{Brotli}} and {{Zstandard}}).

--- middle

# Introduction

This specification defines a mechanism for using designated {{HTTP}} responses
as an external dictionary for future HTTP responses for compression schemes
that support using external dictionaries (e.g. {{Brotli}} and {{Zstandard}}).

This document describes the HTTP headers used for negotiating dictionary usage
and registers media types for content encoding Brotli and Zstandard using a
negotiated dictionary.

# Dictionary Negotiation

## Use-As-Dictionary

When responding to a HTTP Request, a server can advertise that the response can
be used as a dictionary for future requests for URLs that match the pattern
specified in the Use-As-Dictionary response header.

The Use-As-Dictionary response header is a Structured Field {{RFC8941}}
sf-dictionary with values for "match", "expires" and "algorithms".

### match

The "match" value of the Use-As-Dictionary header is a sf-string value that
provides an URL-matching pattern for requests where the dictionary can be used.

The sf-string is parsed as a URL (https://url.spec.whatwg.org/), including
support for parsing URLs relative to the request that provided the
Use-As-Dictionary response. When stored, any relative URLs MUST be expanded
so that only absolute URL patterns are used for matching against requests.

The URL supports using * as a wildcard for pattern-matching multiple URLs.

The "match" value is required and MUST be included in the Use-As-Dictionary
sf-dictionary for the dictionary to be considered valid.

### expires

The "expires" value of the Use-As-Dictionary header is a sf-integer value that
provides the time in seconds that the dictionary is valid for.

This is independent of the cache lifetime of the resource being used for the
dictionary. If the underlying resource is evicted from cache then it is also
removed but this allows for setting an explicit time to live for use as a
dictionary independent of the underlying resource in cache. Expired resources
can still be useful as dictionaries while they are in cache and can be used for
fetching updates of the expired resource. It can also be useful to artificially
limit the life of a dictionary in cases where the dictionary is updated
frequently, to limit the number of possible incoming dictionary values.

The "expires" value is optional and defaults to 31536000 (1 year).

### algorithms

The "algorithms" value of the Use-As-Dictionary header is a inner-list value
that provides a list of supported hash algorithms in order of server
preference.

The dictionaries are identified by the hash of their contents and this value
allows for negotiation of the algorithm to use.

The "algorithms" value is optional and defaults to (sha-256).

### Examples

#### Path Prefix

A response that contained a response header:

Use-As-Dictionary: match="/product/*", expires=604800, algorithms=(sha-256 sha-512)

Would specify matching any URL with a path prefix of /product/ on the same
origin as the original request, expiring as a dictionary in 7 days independent
of the cache lifetime of the resource, and advertise support for both sha-256
and sha-512 hash algorithms.

#### Versioned Directories

A response that contained a response header:

Use-As-Dictionary: match="/app/*/main.js"

Would match main.js in any directory under /app/, expiring as a dictionary in
one year and support using the sha-256 hash algorithm.

## Sec-Available-Dictionary

When a HTTP client makes a request for a resource for which it has an
appropriate dictionary, it can add a "Sec-Available-Dictionary" request header
to the request to indicate to the server that it has a dictionary available to
use for compression.

The "Sec-Available-Dictionary" request header is a Structured Field {{RFC8941}}
sf-string value that contains the hash of the contents of a single available
dictionary calculated using one of the algorithms advertised as being supported
by the server.

### Dictionary URL matching

When a dictionary is stored as a result of a "Use-As-Dictionary" directive, it
includes a "match" string with the URL pattern of request URLs that the
dictionary can be used for.

When comparing request URLs to the available dictionary match patterns, the
comparison should account for the * wildcard when matching against request
URLs. This can be done using regex matching (escape all characters except
for * and replace * with .* to build an appropriate regex).

To be considered as a match, the dictionary must not yet be expired as a
dictionary.

When there are multiple dictionaries that match a given request URL, the client
SHOULD pick the dictionary with the longest match pattern string length.

# Negotiating the compression algorithm

When a compression dictionary is available for use for a given request, the
algorithm to be used is negotiated through the regular mechanism for
negotiating content encoding in HTTP.

This document introduces two new content encoding algorithms:

"sbr" - Brotli using an external compression dictionary.
"szstd" - Zstandard using an external compression dictionary.

The dictionary to use is negotiated separately and advertised in the
"Sec-Available-Dictionary" request header.

## Accept-Encoding

The client adds the algorithms that it supports to the "Accept-Encoding"
request header. e.g.:

Accept-Encoding: gzip, deflate, br, zstd, sbr, szstd

## Content-Encoding

If a server supports one of the dictionary algorithms advertised by the client
and chooses to compress the content of the response using the dictionary that
the client has advertised then it sets the "Content-Encoding" response header
to the appropriate value for the algorithm selected. e.g.:

Content-Encoding: sbr

# IANA Considerations

## Content Encoding

IANA will add the following entries to the "HTTP Content Coding Registry"
within the "Hypertext Transfer Protocol (HTTP) Parameters" registry:

Name: sbr
Description: A stream of bytes compressed using the Brotli protocol with an
external dictionary

Name: szstd
Description: A stream of bytes compressed using the Zstandard protocol with
an external dictionary

## Header Field Registration

# Compatibility Considerations

To minimize the risk of middle-boxes incorrectly processing
dictionary-compressed responses, compression dictionary transport SHOULD only
be used in secure contexts (HTTPS).

# Security Considerations

The security considerations for [Brotli] and [Zstandard] apply to the
dictionary-based versions of the respective algorithms.

## Changing content

The dictionary must be treated with the same security precautions as
the content, because a change to the dictionary can result in a
change to the decompressed content.

## Reading content

The CRIME attack shows that it's a bad idea to compress data from
mixed (e.g. public and private) sources -- the data sources include
not only the compressed data but also the dictionaries. For example,
if you compress secret cookies using a public-data-only dictionary,
you still leak information about the cookies.

Not only can the dictionary reveal information about the compressed
data, but vice versa, data compressed with the dictionary can reveal
the contents of the dictionary when an adversary can control parts of
data to compress and see the compressed size. On the other hand, if
the adversary can control the dictionary, the adversary can learn
information about the compressed data.

## Security Mitigations

### Cross-origin protection

To make sure that a dictionary can only impact content from the same origin
where the dictionary was served, the "match" pattern used for matching a
dictionary to requests SHOULD be for the same origin that the dictionary
is served from.

### Response readability

For clients, like web browsers, that provide additional protection against the
readability of the payload of a response and against user tracking, additional
protections SHOULD be taken to make sure that the use of dictionary-based
compression does not reveal information that would not otherwise be available.

In these cases, dictionary compression SHOULD only be used when both the
dictionary and the compressed response are fully readable by the client.

In browser terms, that means that both are either same-origin to the context
they are being fetched from or that both include an
"Access-Control-Allow-Origin" response header that matches the "Origin" request
header they are fetched from.

# Privacy Considerations

Since dictionaries are advertised in future requests using the hash of the
content of the dictionary, it is possible to abuse the dictionary to turn it
into a tracking cookie.

To mitigate any additional tracking concerns, clients SHOULD treat dictionaries
in the same way that they treat cookies. This includes partitioning the storage
as cookies are partitioned as well as clearing the dictionaries whenever
cookies are cleared.

--- back

# Acknowledgments
{:numbered="false"}
