---
# Syntax: https://github.com/cabo/kramdown-rfc/wiki/Syntax

stand_alone: true

title: "Partial Content Uploads in HTTP"
abbrev: "Partial Content Uploads"
docname: draft-partial-content-uploads-latest
category: info

submissiontype: IETF
number:
date:
consensus: true
v: 3
area: AREA
workgroup: WG Working Group
keyword:
 - partial
 - upload
 - range
 - http
venue:
  github: commonsensesoftware/partial-content-uploads

author:
 -
    ins: C. Martinez
    fullname: Christopher Martinez
    email: chris.s.martinez@outlook.com

normative:
    RFC8594:
    RFC7807:
    RFC7232:
    RFC7232:
    RFC7231:
    RFC7230:
    RFC6266:
    RFC6266:

informative:
    XHR:
        target: https://xhr.spec.whatwg.org/
        title: XMLHttpRequest, Living Standard
        author:
            -
                ins: A. van Kesteren
                name: Anne van Kesteren
                email: annevk@annevk.nl
        date: 27 September 2023

--- abstract

The Hypertext Transfer Protocol (HTTP) is a stateless application-level protocol for distributed, collaborative, hypertext information systems. This document defines partial content uploads, which allows a client to upload content, such as a large file, via multiple requests. This document also outlines the metadata header fields for indicating state changes, request header fields for making preconditions on such state, and the rules for constructing the responses.

--- middle

# Introduction {#introduction}
{::boilerplate bcp14-tagged}

Hypertext Transfer Protocol (HTTP) clients often encounter interrupted data transfers because of canceled requests or dropped connections. RFC 7233 defines HTTP range requests and partial responses, but it only describes how a client can transfer data from a server (e.g. download).

This document defines a reciprocal set of HTTP range requests and responses that a client can use to transfer data to a server (e.g. upload). Where appropriate, this document will use existing range units and header fields defined in {{!RFC7233}}, {{!RFC6266}}, and {{!RFC2183}}.

Although the range request mechanism is designed to allow for extensible range types, this specification only defines requests for byte ranges.

Terminology {#terminology}

{::boilerplate bcp14}

This specification defines conformance criteria for both senders (usually, HTTP user agents) and recipients (usually, HTTP origin servers). An implementation is considered conformant if it complies with all the requirements associated with its role.

>Syntax Notation

# New Partial Content Transfer
{::boilerplate bcp14-tagged}

The POST method MUST be used to indicate that the client intends to start a new partial content transfer. The client MUST send the Content-Disposition header field defined in {{!RFC6266}} to indicate how the origin server should process the content. This will provide enough information for the origin server to allocate the required storage space before any content is transferred. This behavior ensures that the origin server has enough storage space and the client is authorized to upload the content.

If the origin server successfully allocates the required storage, it MUST respond with 201 (Created), including a Location and ETag header field. If an origin server refuses to allocate the requested storage (ex: due to policy limit), it MUST respond with 400 (Bad Request). It is RECOMMENDED that such a response include problem details as defined in {{!RFC 7807}}, which explains why the content cannot be allocated. When an origin server fails to allocate storage for the resource, then it MUST respond with 507 (Insufficient Storage). If the client is not authorized to create the requested resource, the origin server MUST respond with 401 (Unauthorized) if authentication is possible; otherwise, it MUST respond with 403 (Forbidden).

There is no temporal specification as to how long a client can take to transfer all the content ranges. A server MAY choose to implicitly cancel a transfer it deems abandoned due to inactivity after an arbitrary period or after an absolute amount of time has passed. It is RECOMMENDED that an origin server which knows when the transfer will be considered canceled return the Sunset header as defined in {{!RFC 8594}}, which indicates the cancellation date and time. {{cancel-transfer}} describes how a transfer is explicitly canceled.

## The Content-Disposition Header Field
{::boilerplate bcp14-tagged}

Content-Disposition is a REQUIRED header field. In its absence, the origin server MUST respond with 400 (Bad Request). This header field indicates that the origin server should create a new storage resource and of what size.

### The Create Disposition Type
{::boilerplate bcp14-tagged}

The disposition type 'create' is REQUIRED and is meant to instruct the origin server that it should pre-allocate the required storage space, but the request itself does not contain a body that is to be written to storage.

### Disposition Parameter: 'Size'
{::boilerplate bcp14-tagged}

The size parameter is REQUIRED and has the same meaning as defined in Section 2.7 of {{!RFC2183}}. The origin server MUST use this value as the amount of storage to allocate in octets.

If a client does not provide the size parameter or the size is equal to or less than zero, the origin server MUST respond with 411 (Length Required).

### Disposition Parameter: 'Filename'
{::boilerplate bcp14-tagged}

The "filename" and "filename*" parameters are OPTIONAL parameters that have the same meaning defined in Section 4.3 of {{!RFC6266}}. When specified by a client, the origin server MAY use the specified file name in lieu of the resource identifier derived from the request URL.

### Disposition Parameter: 'Modification-Date'
{::boilerplate bcp14-tagged}

The modification-date parameter is OPTIONAL and has the same meaning as defined in Section 2.5 of {{!RFC2183}}. When the client does not specify the modification-date parameter, the current date and time on the origin server MAY be used if the server has a clock.

## Resource Contention
{::boilerplate bcp14-tagged}

It is possible that multiple clients MAY request an origin server provision storage for the same resource. An origin server SHOULD use optimistic behavior such that the last successful client request is the owner of the remaining transfer requests. The entity tag returned by the origin server in the response provides the correlation to all other requests for a given client. If the server is unable to fulfill the request because the resource cannot be provisioned, it MUST respond with 409 (Conflict) to which a client MAY retry the request. A server MAY also return the Retry-After header field as defined in Section 7.1.3 of {{!RFC7231}} to provide a hint to the client as to how long it should wait before retrying the request.

It is NOT RECOMMENDED that origin servers employ a stateful locking strategy as it could result in a condition by which no client is able to create the requested resource. This document does not describe how to lock or unlock such a resource.

## Entity Tags
{::boilerplate bcp14-tagged}

Upon successful allocation of the required storage space, the origin server MUST return an entity tag in the response. The entity tag MUST be used for all subsequent requests, including canceling the entire transfer. All entity tags returned by the origin server SHOULD use strong validation.

## Resource Retrieval
{::boilerplate bcp14-tagged}

An origin server which supports partial content uploads MAY also support requesting that same resource for downloads. An origin server that implements this capability MUST NOT allow GET requests, in part or in whole, to the provisioned resource until all the corresponding resource content has been transferred to the origin server. The origin server MUST respond with 404 (Not Found) for any resource that has not been completely transferred.

## Example
{::boilerplate bcp14-tagged}

Here is an example of a POST request to initiate a new partial content transfer:

~~~~ http
Content-Disposition: create; size=4294967296
~~~~

An example response would be:

~~~~
Location: <URL>
ETag: "sz8L2qGcV0SHqg8rXwALVQ=="
Sunset: Mon, 13 Nov 2023 00:00:00 GMT
~~~~

# Update Partial Content Transfer
{::boilerplate bcp14-tagged}

The PATCH method MUST be used to update partial content. Unlike other uses of PATCH, an origin server MUST complete this operation idempotently. Given the entity tag provided by the client and the fact that the required storage space has already been allocated, the origin server has enough information to safely fulfill the request idempotently. This behavior is true even if multiple client requests occur concurrently or overlap in content range.

If the origin server successfully updates the specified content range, then it MUST respond with 204 (No Content). If the Content-Disposition header field contained the modification-date parameter, then the origin server MUST also return this value in the response Last-Modified header field.

## Content-Range Header Field
{::boilerplate bcp14-tagged}

Content-Range is a REQUIRED header field that has the same meaning as defined in Section 4.2 of {{!RFC7233}}. The Content-Range header field informs the origin server what part of the content is being transferred.

If the byte-range in Content-Range is invalid or the complete-length does not match the provisioned storage amount defined by the size parameter in the Content-Disposition header field when the partial content transfer was started, the server MUST respond with 416 (Range Not Satisfiable). A client SHOULD inspect the complete-length of the Content-Range header field in the error response to determine whether the source content has been modified.

## Last-Modified Header Field
{::boilerplate bcp14-tagged}

Last-Modified is an OPTIONAL header field and has the same meaning as defined in Section 2.2 of {{!RFC7232}}. If the modification-date parameter was specified in the Content-Disposition header field as part of the start of the partial content transfer, then the origin server MUST respond with the Last-Modified header field containing the same value.

A client MAY use the Last-Modified header field to determine whether its copy of the content being transferred is still the same as the copy the origin server has.

## If-Match Header Field {#if-match-header}
{::boilerplate bcp14-tagged}

If-Match is a REQUIRED header field. The If-Match header field MUST contain the entity tag returned by the origin server when the partial content transfer was started. If a client does not specify the If-Match header field, the server MUST respond with 428 (Precondition Required). If the entity tag specified by the client does not match the value known to the server, the server MUST respond with 412 (Precondition Failed).

## Expect Header Field
{::boilerplate bcp14-tagged}

Expect is an OPTIONAL header field and has the same meaning as defined in Section 5.1.1 of {{!RFC7231}}. Partial content uploads can still be large. Clients SHOULD send the 100-Continue expectation to ensure the server is willing to accept the size of the content being sent. If the client sends more partial content than the server is willing to accept in a single request, it MUST respond with 413 (Entity Too Large).

## Resource Contention
{::boilerplate bcp14-tagged}

It is possible that multiple clients MAY request that an origin server update a partial content transfer for the same resource. Provided that the If-Match header field contains a valid entity tag ({{if-match-header}}), an origin server SHOULD use an optimistic behavior when copying the transferred content to storage. If the server is unable to fulfill the request because the resource is inaccessible, it MUST respond with 409 (Conflict). Since the request MUST be idempotent, a client MAY retry a 409 error response. A server MAY also return the Retry-After header field as defined in Section 7.1.3 of {{!RFC7231}} to provide a hint to the client as to how long it should wait before retrying the request.

If the server is unable to fulfill the request because the allocated storage for the resource no longer exists, the server MUST respond with 410 (Gone).

Retrying large content transfers can strain network resources and are likely to have high latency. If the origin server is aware that the storage resource is temporarily unavailable, it is RECOMMENDED that it automatically retry copying the transferred content. The number of times the server retries, or whether that server retries at all, is at the discretion of the server.

## Completing the Transfer
{::boilerplate bcp14-tagged}

To complete a transfer, a client MUST send all the corresponding content ranges. A client MAY vary the size of each transfer; for example, it may increase or decrease the content range based on available network bandwidth. A server knows that the transfer is complete when it has received transfers from a client that contain contiguous content ranges up to the size of the total content, potentially overlapping.

A server SHOULD NOT be stateful. A server, however, MAY choose to store the start and end position of each content range received in external storage such as a file or database. The exact mechanism used to implement this behavior is at the discretion of the server. Once the server has made the decision that all the content has been transferred, it MAY allow access to the resource via the GET method.

## Example

Here is an example of a PATCH request which transfers part of an upload:

~~~~ http
Content-Type: application/octet-stream
Content-Length: 104857600
Content-Range: bytes 0-104857600/4294967296
If-Match: "sz8L2qGcV0SHqg8rXwALVQ=="
Expect: 100-continue

…<message body>…
~~~~

# Cancel Partial Content Transfer {#cancel-transfer}
{::boilerplate bcp14-tagged}

The DELETE method MUST be used to cancel a partial content transfer. When a client requests that a partial content transfer be canceled, the server MUST deallocate the storage previously provisioned for the transfer.

Only the client originator can cancel the content transfer. The same client or a new client MAY begin a new partial content transfer before the existing transfer is complete. Starting a new partial content transfer intrinsically cancels any existing transfer.

## If-Match Header Field
{::boilerplate bcp14-tagged}

If-Match is a REQUIRED header field. The If-Match header field MUST contain the entity tag returned by the origin server when the partial content transfer was started. If a client does not specify the If-Match header field, the server MUST respond with 428 (Precondition Required). If the entity tag specified by the client does not match the value known to the server, the server MUST respond with 412 (Precondition Failed).

## Resource Contention
{::boilerplate bcp14-tagged}

It is possible that multiple clients MAY request that an origin server cancel a partial content transfer for the same resource. If the server is unable to fulfill the request because the resource is inaccessible, it MUST respond with 409 (Conflict). Since the request MUST be idempotent, a client MAY retry a 409 error response. A server MAY also return the Retry-After header field as defined in Section 7.1.3 of {{!RFC7231}} to provide a hint to the client as to how long it should wait before retrying the request.

If the server is unable to fulfill the request because the allocated storage for the resource no longer exists, the server MUST respond with 204 (No Content) because the transfer has already been canceled.

## Example

Here is an example of canceling a transfer:

~~~~ http
If-Match: "sz8L2qGcV0SHqg8rXwALVQ=="
~~~~

# Security Considerations

TODO Security

# IANA Considerations

TODO IANA

--- back

# Appendix A. Content-Disposition Versus Content-Length
{:numbered="false"}

The Content-Length header field defined in Section 3.3.2 of {{!RFC7630}} is the most appropriate header field to indicate the size of the intended content being transferred. As Section 3 of {{!RFC7631}} indicates, a 'representation' can be anything. In this document, the 'representation' would be the allocated storage for the content transfer, such as a file.

The client scripting engines of modern browsers use the XMLHttpRequest (XHR) API. Section 4.5.2 of the {{?XHR}} specification clearly indicates that the Content-Length header is restricted and may only be set by the browser. The inability to set the Content-Length header without a body makes it an unusable header field as it relates to this document.

In lieu of defining a new header field, this document elected to use the existing Content-Disposition header field defined in {{!RFC2183}} and {{!RFC6266}} to serve the same purpose.