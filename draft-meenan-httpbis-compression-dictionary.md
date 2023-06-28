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
Use-As-Dictionary response.

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

## Advertising Dictionary Availability

### Dictionary URL matching

Wildcard support

# Content Encoding

# IANA Considerations

## Header Field Registration

## Content Encoding

# Compatibility Considerations

To minimzide the risk of middle-boxes incorrectly processing dictionary-compressed
responses, compression dictionary transport SHOULD only be used in secure contexts
(HTTPS).

# Security Considerations

# Privacy Considerations

--- back

# Acknowledgments
{:numbered="false"}
