---
# Syntax: https://github.com/cabo/kramdown-rfc/wiki/Syntax

stand_alone: true

title: "Partial Content Uploads in HTTP"
abbrev: "Partial Content Uploads"
docname: draft-ietf-partial-content-uploads-latest
category: info
ipr: trust200902
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
    RFC9110:
    RFC8594:
    RFC7807:
    RFC6266:
    RFC5234:
    RFC3864:
    RFC2183:

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
    FETCH:
        target: https://fetch.spec.whatwg.org/
        title: Fetch Standard
        author:
            -
                org: Web Hypertext Application Technology Working Group
                abbrev: WHATWG
                uri: https://github.com/whatwg/fetch
        date: 25 March 2024

--- abstract

The Hypertext Transfer Protocol (HTTP) is a stateless application-level protocol for distributed, collaborative, hypertext information systems. This document defines partial content uploads, which allows a client to upload content, such as a large file, via multiple requests. This document also outlines the metadata header fields for indicating state changes, request header fields for making preconditions on such state, and the rules for constructing the responses.

--- middle

# Introduction {#introduction}

Hypertext Transfer Protocol (HTTP) clients often encounter interrupted data transfers because of canceled requests or dropped connections. Section 14 of {{!RFC9110}} defines HTTP range requests and partial responses, but it only describes how a client can transfer data from a server (e.g. download).

This document defines a reciprocal set of HTTP range requests and responses that a client can use to transfer data to a server (e.g. upload). Where appropriate, this document will use existing range units and header fields defined in {{!RFC9110}}, {{!RFC6266}}, and {{!RFC2183}}.

Although the range request mechanism is designed to allow for extensible range types, this specification only defines requests for byte ranges.

## Terminology {#terminology}

{::boilerplate bcp14}

This specification defines conformance criteria for both senders (usually, HTTP user agents) and recipients (usually, HTTP origin servers). An implementation is considered conformant if it complies with all the requirements associated with its role.

## Syntax Notation

This specification uses the Augmented Backus-Naur Form (ABNF) notation of {{!RFC5234}} with a list extension, defined in Section 2.1 of {{!RFC9110}}, that allows for compact definition of comma-separated lists using a '#' operator (similar to how the '*' operator indicates repetition).

# New Partial Content Upload {#new-upload}

The POST method MUST be used to indicate that the client intends to start a new partial content upload. The client MUST send the Content-Disposition header field defined in {{!RFC6266}} to indicate how the origin server is expected process the content. This will provide enough information for the origin server to allocate the requested storage space before any content is uploaded. This behavior ensures that the origin server has enough storage space and the client is authorized to upload the content.

If the origin server successfully allocates the necessary storage, it MUST respond with 201 (Created), including a Location and ETag header field. A server MAY elect to return the Allow-Length header field, which indicates the maximum allowed length of any subsequent partial content. If an origin server refuses to allocate the requested storage (ex: due to policy limit), it MUST respond with 422 (Unprocessable Content). It is RECOMMENDED that such a response include problem details as defined in {{!RFC7807}} which explains why the content cannot be allocated. When an origin server fails to allocate storage for the resource, then it MUST respond with 507 (Insufficient Storage). If the client is not authorized to create the requested resource, the origin server MUST respond with 401 (Unauthorized) if authentication is possible; otherwise, it MUST respond with 403 (Forbidden).

There is no temporal specification as to how long a client is allowed take to upload all the content ranges. A server MAY choose to implicitly cancel an upload it deems abandoned due to inactivity after an arbitrary period or after an absolute amount of time has passed. It is RECOMMENDED that an origin server which knows when the upload will be considered canceled return the Sunset header as defined in {{!RFC8594}}, which indicates the cancellation date and time. {{cancel-upload}} describes how an upload is explicitly canceled.

## The Content-Disposition Header Field

Content-Disposition is a REQUIRED header field. In its absence, the origin server MUST respond with 400 (Bad Request). This header field indicates that the origin server is to create a new storage resource and of what size.

### The Create Disposition Type {#create-disposition-type}

The disposition type "create" is REQUIRED and is meant to instruct the origin server that it SHOULD preallocate the specified storage space, but the request itself does not contain a body that is to be written to storage.

If a client were to send one of the existing disposition types "inline" or "attachment" to an origin server without a body, it is not clear what action the server MUST take. The disposition type "create" expresses a
client's expectation that the disposition of the origin server is to create a resource of "size" octets.

### Disposition Parameter: 'Size'

The size parameter is REQUIRED and has the same meaning as defined in Section 2.7 of {{!RFC2183}}. The origin server MUST use this value as the amount of storage to allocate in octets.

If a client does not provide the size parameter or the size is equal to or less than zero, the origin server MUST respond with 411 (Length Required). If the origin server refuses to allocate the requested storage, it MUST
respond with 422 (Unprocessable Content). It is RECOMMENDED that such a response include problem details as defined in {{!RFC7807}} which explains why the content cannot be allocated and includes the maximum allowable size. For example,
if the requested size is too large, the server MAY respond with:

~~~~ http
Content-Type: application/problem+json
Content-Language: en

{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.1",
  "title": "The requested resource size is too large."
  "detail": "1TB exceeds the 200GB maximum, allowed size.",
  "max-size": 2e+11
}
~~~~

### Disposition Parameter: 'Filename'

The "filename" and "filename*" parameters are OPTIONAL parameters that have the same meaning defined in Section 4.3 of {{!RFC6266}}. When specified by a client, the origin server MAY use the specified file name in lieu of the resource identifier that it elects to generate, which MAY also derive from the request URL.

### Disposition Parameter: 'Modification-Date'

The modification-date parameter is OPTIONAL and has the same meaning as defined in Section 2.5 of {{!RFC2183}}. When the client does not specify the modification-date parameter, the current date and time on the origin server MAY be used if the server has a clock.

## The Allow-Length Header Field {#allow-length}

The Allow-Length response header field allows a server to communicate the allowable length of content. It provides information to clients that enables them to delimit framing as they send partial content to the server. It is RECOMMENDED that an origin server return the Allow-Length header field when a resource is provisioned for an upload.

~~~~
Content-Length = 1*DIGIT
~~~~

An example is

~~~~ http
Allow-Length: 10000000
~~~~

In the absence of the Allow-Length header field, a client is obligated to know the maximum length allowed through out-of-band knowledge such as a publicly documented policy or through probing requests until a suitable length is determined.

An origin server MAY choose to return Allow-Length in a HEAD or OPTIONS request for a client that did not persist the value and resumes an upload at a later time. The specified Allow-Length SHOULD NOT change for the lifetime of an upload. If the value does change during an upload, then the origin server SHOULD support the HEAD method, OPTIONS method, or both, that SHALL respond with an updated Allow-Length header field.

## Resource Contention

It is possible that multiple clients MAY request an origin server provision storage for the same resource. An origin server SHOULD use optimistic behavior such that the last successful client request is the owner of the remaining upload requests. The entity tag returned by the origin server in the response provides the correlation to all other requests for a given client. If the server is unable to fulfill the request because the resource cannot be provisioned, it MUST respond with 409 (Conflict) to which a client MAY retry the request. A server MAY also return the Retry-After header field as defined in Section 10.2.3 of {{!RFC9110}} to provide a hint to the client as to how long it SHOULD wait before retrying the request.

It is NOT RECOMMENDED that origin servers employ a stateful locking strategy as it could result in a condition by which no client is able to create the requested resource. This document does not describe how to lock or unlock such a resource.

## Entity Tags

Upon successful allocation of the requested storage space, the origin server MUST return an entity tag in the response. The entity tag MUST be used for all subsequent requests, including canceling the entire upload. All entity tags returned by the origin server SHOULD use strong validation.

## Resource Retrieval

An origin server which supports partial content uploads MAY also support requesting that same resource for downloads. An origin server that implements this capability MUST NOT allow GET requests, in part or in whole, to the provisioned resource until all the corresponding resource content has been uploaded to the origin server. The origin server MUST respond with 404 (Not Found) for any resource that has not been completely uploaded.

## Example

Here is an example of a POST request to initiate a new partial content upload:

~~~~ http
Content-Disposition: create; size=4294967296
~~~~

An example response would be:

~~~~
Allow-Length: 10000000
ETag: "sz8L2qGcV0SHqg8rXwALVQ=="
Location: <URL>
Sunset: Mon, 13 Nov 2023 00:00:00 GMT
~~~~

# Update Partial Content Upload {#update-upload}

The PATCH method MUST be used to update partial content. Unlike other uses of PATCH, an origin server MUST complete this operation idempotently. Given the entity tag provided by the client and the fact that the necessary storage space has already been allocated, the origin server has enough information to safely fulfill the request idempotently. This behavior is true even if multiple client requests occur concurrently or overlap in content range.

If the origin server successfully updates the specified content range and more content is expected, then it MUST respond with 202 (Accepted); otherwise, the server MUST complete the upload as described in {{complete-upload}}. If the Content-Disposition header field contained the modification-date parameter, then the origin server MUST also return this value in the response Last-Modified header field.

## Content-Range Header Field

Content-Range is a REQUIRED header field that has the same meaning as defined in Section 14.4 of {{!RFC9110}}. The Content-Range header field informs the origin server what part of the content is being uploaded.

If the byte-range in Content-Range is invalid or the complete-length does not match the provisioned storage amount defined by the size parameter in the Content-Disposition header field when the partial content upload was started, the server MUST respond with 416 (Range Not Satisfiable). A client SHOULD inspect the complete-length of the Content-Range header field in the error response to determine whether the source content has been modified.

## Last-Modified Header Field

Last-Modified is an OPTIONAL header field and has the same meaning as defined in Section 8.8.2 of {{!RFC9110}}. If the modification-date parameter was specified in the Content-Disposition header field as part of the start of the partial content upload, then the origin server MUST respond with the Last-Modified header field containing the same value.

A client MAY use the Last-Modified header field to determine whether its copy of the content being uploaded is still the same as the copy the origin server has.

## If-Match Header Field {#if-match-header}

If-Match is a REQUIRED header field. The If-Match header field MUST contain the entity tag returned by the origin server when the partial content upload was started. If a client does not specify the If-Match header field, the server MUST respond with 428 (Precondition Required). If the entity tag specified by the client does not match the value known to the server, the server MUST respond with 412 (Precondition Failed).

## Expect Header Field

Expect is an OPTIONAL header field and has the same meaning as defined in Section 10.1.1 of {{!RFC9110}}. It is still possible for partial content uploads to be large. Clients SHOULD send the 100-Continue expectation to ensure that the server is willing to accept the size of the content being sent. If the client sends more partial content than the server is willing to accept in a single request, it MUST respond with 413 (Payload Too Large). The server SHOULD also respond with the Allow-Length header field ({{allow-length}}) to indicate the maximum length allowed by the server.

## Resource Contention

It is possible that multiple clients MAY request that an origin server update a partial content upload for the same resource. Provided that the If-Match header field contains a valid entity tag ({{if-match-header}}), an origin server SHOULD use an optimistic behavior when copying the uploaded content to storage. If the server is unable to fulfill the request because the resource is inaccessible, it MUST respond with 409 (Conflict). Since the request MUST be idempotent, a client MAY retry a 409 error response. A server MAY also return the Retry-After header field as defined in Section 10.2.3 of {{!RFC9110}} to provide a hint to the client as to how long it SHOULD wait before retrying the request.

If the server is unable to fulfill the request because the allocated storage for the resource no longer exists, the server MUST respond with 410 (Gone).

Retrying large content uploads MAY strain network resources and are likely to have high latency. If the origin server is aware that the storage resource is temporarily unavailable, it is RECOMMENDED that it automatically retry copying the uploaded content. The number of times the server retries, or whether that server retries at all, is at the discretion of the server.

## Completing the Upload {#complete-upload}

To complete an upload, a client MUST send all the corresponding content ranges. A client MAY vary the size of each upload; for example, it MAY increase or decrease the content range based on available network bandwidth. A server knows that the upload is complete when it has received uploads from a client that contain all of the contiguous content ranges up to the size of the total content, potentially overlapping. If the origin server determines no further content is expected, then it MUST respond with 201 (Created) and Content-Location. Content-Location indicates the URL where the client MAY retrieve the resource with a GET request, which MAY not previously be known to the client.

A server SHOULD NOT be stateful. A server, however, MAY choose to store the start and end position of each content range received in external storage such as a file or database. The exact mechanism used to implement this behavior is at the discretion of the server. Once the server has made the decision that all of the content has been uploaded, it MAY allow access to the resource via the GET method that is indicated in Content-Location.

## Example

Here is an example of a PATCH request which uploads part of an upload:

~~~~ http
Content-Type: application/octet-stream
Content-Length: 104857600
Content-Range: bytes 0-104857600/4294967296
If-Match: "sz8L2qGcV0SHqg8rXwALVQ=="
Expect: 100-continue

…<message body>…
~~~~

# Cancel Partial Content Upload {#cancel-upload}

The DELETE method MUST be used to cancel a partial content upload. When a client requests that a partial content upload be canceled, the server MUST deallocate the storage previously provisioned for the upload.

Only the client originator MAY cancel the content upload. The same client or a new client MAY begin a new partial content upload before the existing upload is complete. Starting a new partial content upload intrinsically cancels any existing upload.

## If-Match Header Field

If-Match is a REQUIRED header field. The If-Match header field MUST contain the entity tag returned by the origin server when the partial content upload was started. If a client does not specify the If-Match header field, the server MUST respond with 428 (Precondition Required). If the entity tag specified by the client does not match the value known to the server, the server MUST respond with 412 (Precondition Failed).

## Resource Contention

It is possible that multiple clients MAY request that an origin server cancel a partial content upload for the same resource. If the server is unable to fulfill the request because the resource is inaccessible, it MUST respond with 409 (Conflict). Since the request MUST be idempotent, a client MAY retry a 409 error response. A server MAY also return the Retry-After header field as defined in Section 10.2.3 of {{!RFC9110}} to provide a hint to the client as to how long it SHOULD wait before retrying the request.

If the server is unable to fulfill the request because the allocated storage for the resource no longer exists, the server MUST respond with 204 (No Content) because the upload has already been canceled.

## Example

Here is an example of canceling an upload:

~~~~ http
If-Match: "sz8L2qGcV0SHqg8rXwALVQ=="
~~~~

# Get Partial Content Upload Metadata

There are situations where a client does not have the necessary information to continue uploading content. A few of these scenarios include a client losing network connectivity, pausing or suspending uploads, and resuming or retrying an upload where the previously uploaded content ranges were not persisted by the client. These scenarios MAY lead a client to request from the destination origin server what information has been already uploaded to it.

The HEAD method MUST be used to request the metadata about a partial content upload. If the partial content upload metadata is successfully retrieved, the server MUST respond with 204 (No Content). If the server is unable to fulfill the request because the resource no longer exists and the server was previously aware of the resource, the server SHOULD respond with 410 (Gone). The origin server MUST, otherwise, respond with 404 (Not Found) for any resource that does not exist.

## Allow-Length Header Field

Allow-Length is an OPTIONAL response header field. It informs the client what the current content size limit is for any partial upload, which MAY be different from the limit reported in any previous response. It is RECOMMENDED that an origin server return the Allow-Length header field with an appropriate value, even when a limit had not been previously set.

## Content-Length Header Field

Content-Length is a REQUIRED response header field. The Content-Length value MUST correspond to the size parameter specified in the original Content-Disposition header field when the partial content upload was created.

## ETag Header Field

ETag is a REQUIRED response header field. The ETag value MUST be the same value generated by the server when the partial content upload was created.

## Last-Modified Header Field

Last-Modified is an OPTIONAL response header field. The Last-Modified header field MUST be returned if the modification-date parameter was specified in the original Content-Disposition header field when the partial content upload was created.

## Sunset Header Field

Sunset is an OPTIONAL response header field. If the server imposes a temporal limit on how much time a client has before it considers the upload abandoned, the server MUST return the Sunset header field containing the absolute date and time that will occur. Any date and time returned by the server SHOULD be the same value returned when the partial content upload was created.

## Range Header Field

Range is an OPTIONAL response header field. The server indicates what content the client has already uploaded by sending the Range header field as defined in Section 14.2 of {{!RFC9110}}. Each range specified MUST use the byte-range unit type as defined by Section 14.1.2 of {{!RFC9110}} with the exception that the first-pos and last-pos values MUST both be specified. A server MAY chose to emit a Range header field for each byte offset range or collapse them into multiple header field values. A server MAY also merge overlapping ranges in order to reduce the number of reported range values.

As an example, if a client previously made 3 partial uploads and with overlapping ranges, the server can emit:

~~~~ http
Range: bytes=0-999999
Range: bytes=900000-1999999
Range: bytes=2000000-2999999
~~~~

Or:

~~~~ http
Range: bytes=0-2999999
~~~~

## Example

Here is an example of a response to a HEAD request which reports the metadata for a partial content upload that is in progress:

~~~~ http
Allow-Length: 10000000
Content-Length: 4294967296
ETag: "sz8L2qGcV0SHqg8rXwALVQ=="
Last-Modified: Sun, 5 Nov 2023 00:00:00 GMT
Range: bytes=0-9999999, 10000000-19999999
Range: bytes=20000000-49999999
Sunset: Mon, 13 Nov 2023 00:00:00 GMT
~~~~

# Security Considerations

This document has the same security considerations that are outlined in Section 5 of {{!RFC2183}} and Section 7 of {{!RFC6266}}. There are additional risks from clients requesting unintentionally large resources. {{new-upload}} and {{update-upload}}
describe the validation process and how to reject such a request. The maximum sizes allowed, in whole or in part, are at the discretion of the origin server.

While this document mandates that entity tags be used in requests, it does not dictate the format or content of those values. An origin server SHOULD generate an entity tag that cannot be easily replayed. A few possible techniques might
include rotating the entity tag during each request as well as encrypting or signing a value in the entity tag using a client certificate. These protective measures would be in addition to transport-level security, client authentication,
and client authorization.

# IANA Considerations

## The Create Disposition Type

This document registers a new "disposition-type" value for the Content-Disposition header: create. The definition and usage of this value is described in {{create-disposition-type}}. IANA is asked to add a new "disposition-type" value to
the Content-Disposition header as defined by {{!RFC2183}}:

create: allocate a resource of "size" octets without a body

The "size" parameter is REQUIRED. The "filename", "creation-date", "modification-date", and "read-date" are OPTIONAL. Any other parameters are unused and SHOULD be ignored.

## The Allow-Length Response Header Field

This document requests that the Allow-Length response header field be added to the "Permanent Message Header Field Names" registry (see {{!RFC3864}}), taking into account the guidelines given by {{!RFC9110}}.

Header Field Name: Allow-Length

Protocol: http

Status: informational

Author/Change controller: IETF

--- back

# Appendix A. Content-Disposition Versus Content-Length
{:numbered="false"}

The Content-Length header field defined in Section 8.6 of {{!RFC9110}} is the most appropriate header field to indicate the size of the intended content being uploaded. As Section 8 of {{!RFC9110}} indicates, a "representation" can be anything. In this document, the "representation" would be the allocated storage for the content to be uploaded, such as a file.

The client scripting engines of modern browsers use the XMLHttpRequest (XHR) API and Fetch Standard. Section 4.5.2 in the {{XHR}} specification and Section 2.2.2 in the {{FETCH}} specification both state that the Content-Length header is restricted and is only allowed to be set by the browser. The inability to set the Content-Length header without a body makes it an unusable header field as it relates to this document.

In lieu of defining a new header field, this document elected to use the existing Content-Disposition header field defined in {{!RFC2183}} and {{!RFC6266}} to serve the same purpose.

# Acknowledgements
{:numbered="false"}

The author would like to thank James Zimmerman, Hongtao Chen, and Xavier John for their valuable contributions and reviews.
