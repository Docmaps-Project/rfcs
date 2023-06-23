# Docmaps Server API Protocol

## Abstract

This proposal is for a standard API contract for servers which offer Docmaps to interoperate with each other. This includes
defining the endpoints that should be implemented for identifying the server, serving docmaps, searching for docmaps, including
rules such as what endpoint results must be stable/cacheable and which may be dynamic.

This proposes a server specification that conforms to W3C Linked Data Platform recommendations, because Docmaps data are
natively considered to be Linked Data. Although an implementer of this protocol may not offer full linked data support,
such as allowing `content-type: text/turtle`, and may not represent the data as a graph or triplestore in their persistent
state solution, they are expected to implement this API at a minimum according to the usages of JSON-LD as described herein,
in order to enable consumers of this data to seamlessly treat it as LD.

## Motivation

At this stage in the development of the Docmaps ecosystem, there is significant demand for Docmaps data for preprints but
relatively little adoption by preprint repositories. Already various preprint repositories are implementing their own
mechanisms for serving docmaps, which is encouraged. However, there is risk of divergent standards emerging that place
significant burden on consumers to handle multiple API contracts. There is additional engineering overhead involved in
building several different client and server libraries in parallel. With a majority of use cases covered by a single
protocol, we can implement the protocol once per language and share the code; and clients written in various languages will
naturally be interoperable.

Additionally, beyond simply the vocabulary of endpoints and their responses, there is no consensus among current technical
stakeholders as to what objects in a Docmaps data ecosystem are permanent or ephemeral, what should have integrity verification
incorporated, and how to handle invalidation. In order to adequately describe the API contract for the initial Docmaps
standard, this proposal answers several of these questions.

A server that issues docmaps which conforms to this protocol will have an easier time acquiring users and consumers,
supporting them, and interacting with other servers.

## Goals

This proposal's protocol will be implemented in the Docmaps Project's reference implementation and will be used
to serve docmaps from Pubpub. I would like to introduce minimal burden on existing Docmaps servers by pattern-matching
to their API designs where it is possible and sustainable for new users as well.

This proposal's scope DOES include rules for:

- when/which docmap queries may return unstable results, and when they must not
- basic trust management
- readonly queries for serving docmaps and related data to consumers.

This proposal's scope does NOT include rules for:

- writeable endpoints/modifying a docmaps datastore.
- strategies for authorizing client requests
- federating servers / pass-through trust management
- authentication or integrity data internal to a docmap's contents
- computation performed on a docmap's contents

## Proposed Solution

This proposal describes a version 1.x. Versioning of this API protocol follows the semver
"semantic versioning" scheme.

All endpoints MAY include in their responses a key
`"protocol_version": "1.0"` to indicate their compatibility. No other 

The endpoints expected of a standard docmaps server are grouped here by use-case.

Servers MAY offer these endpoints at a path prefix which is any of the following:

```
/v1/
/api/v1/
**/docmaps/v1
```

See `/info` below.

Implementers of this protocol SHOULD expect that the meaning of any field will be stable for any future
versions. , and backwards-incompatible
changes will result in a path change to `*/v2` and above.

All endpoints that return body contents MUST respond with JSON. Where indicated, they MAY respond with JSON-LD.

This proposes a server specification that conforms to [W3C Linked Data Platform (LDP) recommendations](https://www.w3.org/TR/2015/REC-ldp-20150226/), because Docmaps data are
natively considered to be Linked Data. Although an implementer of this protocol may not offer full linked data support,
such as allowing `content-type: text/turtle`, and may not represent the data as a graph or triplestore in their persistent
state solution, they are expected to implement this API at a minimum according to the usages of JSON-LD as described herein,
in order to enable consumers of this data to seamlessly treat it as LD.

Where this recommendation may disagree or be incompatible with the LDP recommendations, implementers are advised to prefer this
document, and open an issue on GitHub to discuss bringing this specification into compliance with LDP.

Any resource path MAY support pagination. If pagination is present, it MUST conform to the [LDP Paging recommendation](https://www.w3.org/TR/ldp-paging/).

### Identifying the server

These endpoints are REQUIRED for all standard Docmaps servers. They enable consumers to make decisions about the identity
and trustworthiness of the server, and manage changes that may occur in the identity profile of the server.

#### GET /info
This endpoint provides information about the server to aid with future utilization:


```json
{
  "api_url": "https://example.com/ex/docmaps/v1/",
  "api_version": "1.0.0",
  "ephemeral_document_expiry": {
    "max_seconds": 60,
    "max_retrievals": 1
  },
  "peers": [
    {
      "api_url": "https://otherserver.com/docmaps/v1"
    }
  ]
}
```

where

| PATH | DETAILS |
| --- | --- |
| `api_url` | The URL with path prefix to this server. It SHOULD end with a slash. It MUST conform to the endpoint URL constraints (see above). |
| `api_version` | MUST be exactly the version of this document (`"1.0.0"`). |
| `ephemeral_document_expiry` | MUST be an object that informs the user about conditions in which ephemeral documents stop becoming available. If this server has conditions in which it expires these documents, it MUST specify those here. If no guarantees are available and documents may expire immediately, it SHOULD specify all expiry indicators as `0`. This property is applicable to search results, including Docmaps for a given DOI, which are implicitly ephemeral. |
| `ephemeral_document_expiry.max_seconds` : Optional. Clients should expect documents to expire after this duration in seconds. |
| `ephemeral_document_expiry.max_retrievals` : Optional. Clients should expect documents to expire after this number of retrievals. |
| `peers` | Optional field to refer clients to other Docmaps servers that may be of interest. If present, MUST be an array of objects, and `peers[].api_url` MUST be a URL with path prefix that directs to a server that conforms to this Protocol. Clients MUST decide on their own whether to trust these other servers. Future updates to this protocol will include how to specify trust information as part of this annotation. |

#### {\*} /trust/\*

Reserved for future use. Servers MUST NOT implement any behavior under these paths except to return 404.

### Querying a known object IRI

These endpoints are REQUIRED for all standard Docmaps servers. They enable consumers to retrieve objects in full which
the server can authoritatively or nonauthoritatively attest.

#### GET /docmap/{ID}

MUST return a JSON body that is a decodable Docmap. MAY include a `@context` key
for users to interpret as a JSON-LD object. MUST NOT return a top-level `@graph`
key. Note that this endpoint may be implemented correctly with any naming scheme for
docmap objects, but consumers should not assume these IDs can be guessed. See "Alternatives" below.

```json
{
   "@context": "https://w3id.org/docmaps/context.jsonld",
   "id": "https://example.com/ex/docmaps/v1/docmap/deadbeef-f1e7-4b2c-a6f5-d8c2f9f9a2e2",
   "type": "docmap",
   "publisher": {
     "name": "John Doe",
     "id": "https://example.com/ex/docmaps/v1/publisher/example_press"
   },
   "created": "2020-01-01",
   "steps": {
     "step-1": {
       "actions": [{
         "outputs": [{
           "published": "2020-01-01",
           "id": "123456",
           "doi": "10.12345/abcdef",
           "type": "Article",
           "content": [{
             "type": "text",
             "text": "This is an example of a thing"
           }]
         }],
         "participants": [{
           "actor": {
             "type": "person",
             "name": "John Doe"
           },
           "role": "author"
         }],
         "id": "123456"
       }],
       "inputs": [{
         "published": "2020-01-01",
         "id": "123456",
         "doi": "10.12345/abcdef",
         "type": "Article",
         "content": [{
           "type": "text",
           "text": "This is an example of a thing"
         }]
       }],
       "assertions": [{
         "item": {
           "type": "Article",
           "id": "123456"
         },
         "status": "accepted",
         "happened": "2020-01-01"
       }],
       "id": "123456",
       "next-step": "step-2"
     },
     "step-2": {
       ...
     }
   },
   "first-step": "step-1",
   "updated": "2020-01-01"
}
```

where

PATH | DETAILS
| - | - |
| `@context` |  If present, MUST be a JSON-LD context. SHOULD be exactly `"https://w3id.org/docmaps/context.jsonld"`  |
| `id` |  The ID of the docmap. MUST be an IRI and MUST be exactly the URL queried to retrieve this docmap.  |

#### GET /publisher/{ID}

Returns information about Docmaps publishers. This endpoint is REQUIRED for servers which
claim to be authoritative publishers of any Docmaps, i.e., if `GET /docmap/{id}` returns any
`publisher.id` which has as a prefix the `api_url` of this server. This endpoint MUST return
valid results for each of those IDs.

If present, MUST return a JSON body that is a decodable Docmap Publisher. MAY include a
`@context` key for users to interpret as a JSON-LD object. MUST NOT return a top-level
`@graph` key.

```json
 {
   "@context": "https://w3id.org/docmaps/context.jsonld",
   "name": "John Doe",
   "id": "https://example.com/ex/docmaps/v1/publisher/example_press",
   "homepage": "https://example.com/",
   "logo": "https://example.com/logo.img",
   "account": {
     "id": "https://github.com/docmaps-project",
     "service": "https://github.com"
   }
 },
```

where

PATH | DETAILS
| - | - |
| `@context` |  If present, MUST be a JSON-LD context. SHOULD be exactly `"https://w3id.org/docmaps/context.jsonld"`  |
| `id` |  The ID of the publisher. MUST be an IRI and MUST be a URL which when queried responds with this publisher exactly. |

Publishers are always named nodes (nodes with IRIs for named nodes (nodes with IRIs) in the Docmaps specification. Therefore
they are always considered persistent. Any signatory information is separate from  their RDF bodies (the data in this  endpoint's
response), so those bodies can safely  change. However, if the IRI of a publisher changes, clients will need to handle the migration
by their own logic until  the core Docmaps protocol is improved ([see Github  Issue here](https://github.com/Docmaps-Project/docmaps/issues/69)).
That  improvement is out of the scope of this RFC.

The authors of this specification anticipate the relationship between a server and specific Publishers whose signing identities
are owned by the server may be expressed in extensions to this endpoint. See "Security" below.

### Searching for docmaps based on content matter

These endpoints are REQUIRED for all standard Docmaps servers. They enable consumers to discover what docmaps or objects
can be retrieved from this server. They return information about further quueries that can be performed to retrieve docmaps.

#### POST /search

The search query MUST be accepted in the request body as type `application/json`. Although servers are invited to
decide on search semantics that are most appropriate to their use case, servers MUST support the following body
fields in the search:

```json
{
  "query_terms": [
    {
      "match": "https://data-hub-api.elifesciences.org/enhanced-preprints/docmaps/v1/index",
      "paths": ["publisher.id"]
    },
    {
      "match": "docmap",
      "paths": ["type"]
    }
  ]
}
```

where

PATH | DETAILS
| - | - |
| `query_terms` | MUST be an array of object. |
| `query_terms[].match` | MUST be a full IRI or a JSON-LD shortcut present in the Docmaps JSON-LD context. |
| `query_terms[].paths` | MUST be an array of JSON paths, IRIs or JSON-LD shortcuts that the match term matches to. |

**Response** MUST return a linked-data (JSON-LD) graph, because this endpoint is not strictly typed.
For use-cases where JSON-LD is of limited applicability, the graph may be disjoint (i.e.,
an array of objects whose contents are unrelated to each other.)

```json
{
  "@context": "https://w3id.org/docmaps/context.jsonld",
  "@graph": [
    {
      "type": "docmap",
      "id": "https://example.com/ex/docmaps/v1/docmap/deadbeef-f1e7-4b2c-a6f5-d8c2f9f9a2e2"
    }
  ]
}
```

where

PATH | DETAILS
| - | - |
| `@context` |  If present, MUST be a JSON-LD context. SHOULD be exactly `"https://w3id.org/docmaps/context.jsonld"`  |
| `@graph` |  MUST be an array of objects.  |
| `@graph[]` |  Each entity MAY be an entity that can be independently decoded into a valid object in Docmaps vocabulary. However, clients MAY be required to retrieve the majority of entity details via followup requests. |
| `@graph[].type` |  SHOULD be an entity type or type IRI allowed in the docmaps vocabulary. Consumers will use this key to identify useable elements in the response body. |
| `@graph[].id` | MUST be an IRI. SHOULD be an IRI to an object that can be served by this API. |

### Convenience endpoints for one-shot noninteractive docmap retrieval

These endpoints enable a consumer with knowledge of a specific document to request further data about
that document in the form of a docmap from this server without first performing a search. These endpoints
are not required, but if they exist, servers MUST implement them according to the protocol specification.
They are included in the specification to reduce adoption overhead for servers that currently have similar
one-shot API endpoints that their clients wish to continue using.

Critically, this endpoint differs from the `/docmap/{ID}`endpoint because with this endpoint, a client MUST
NOT expect that  the body's `id` contains an IRI matching this endpoint. For this reason, this endpoint
should be considered non-canonical. Note that the subject DOI is provided as a query parameter rather than a
path parameter.

#### GET /docmap_for/doi?subject={doi}
#### GET /docmap_for/iri?subject={iri}

**Response** matching response body structure of `/docmap/{ID}

### Batch queries for Indexing

These endpoints are OPTIONAL for Docmaps servers that wish to support being indexed. They allow a client to synchronize
its application state with the latest state of the server's dataset. This use-case has stricter requirements for data
integrity on the server's internal state management than the other use-cases.

#### GET /synchronization*?from=9999&limit=9999&session=ff000000*

This endpoint exposes a sync-forward API for linked-data consumers to ingest wholesale the contents
of an RDF graph that stores docmaps. This is the preferred method of indexing. Note that while the
specification of this endpoint allows an implementer to respond purely with lists of Docmaps, it is
preferred to respond with persistent objects only, and allow the indexer to construct docmaps dynamically.

This endpoint is designed to allow servers to decide what constitutes a sync-head. For event-backed datasets, it may be
a serial ID generated for each sequential event. For datasets that support sync via point-in-time snapshots, the sync
head may be per user.

This endpoint MUST handle parameters supplied as URL Query Parameters, and SHOULD NOT support accepting these
parameters as fields in the request body.

- from: a sync head. if present, must be number, can be timestamp or preferably a serial. if omitted, start from earliest known.
Conventionally, element(s) that match `from` are included. Timestamp implementations of `from` may have unspecified behavior
and are not recommended.
- limit: a specified maximum number of elements. Because Docmaps data are natively graph based, the semantics of this
limit are left to implementers and SHOULD NOT be relied upon by clients to produce predictable behavior. A given server
MAY encourage consumers of the API to use this parameter to intentfully specify their capacity for pagination. A client
MUST NOT assume that all servers will handle this parameter in the same way.
- session: OPTIONAL, any string identifier that the server may require for purposes of tracking session-based syncs. The server
may decide whether this field is required and any further substructure expected within the string that their session cardinality
may depend on. Note that session identification relates  to client identification  and therefore there are security considerations.
For example, a server may issue a session token on request for a sync with no `from` specified, as starting a new sync
session. This session token can be used to disrupt the sync session and so is considered a secret, even though the impact of
impersonation may not be severe.

**Response** MUST respond with an LD-graph  of objects that may be disjoint. These objects MUST all have
IRIs in their `ID` field.

Note that this endpoint is asssumed to be dependent on pagination, and MUST conform to LDP Paging (see above).

The `next` `Link` response header is REQUIRED for this endpoint. If that link refers to a next page which does not yet exist,
because the synchronizing client has caught up, it MUST serve `HTTP 202 Accepted` response. Clients are expected to resubmit
the same query later according to a reasonable back-off scheme.

```http
GET https://example.com/docmaps/v1/synchronization?from=9999&limit=9999&session=ff000000

HTTP/1.1 200 OK
Content-Type:  application/ld+json
Link: <https://example.com/docmaps/v1/synchronization?from=19998&limit=9999&session=ff000000>; rel="next"
```

```json
{
   "@context": "https://w3id.org/docmaps/context.jsonld",
   "@graph": [
      {
        "type": "docmap",
        "id": "https://example.com/ex/docmaps/v1/docmap/deadbeef-f1e7-4b2c-a6f5-d8c2f9f9a2e2"
      }
   ]
}
```

where

| Field | Value |
| -------- | -------- |
| `@context` |  If present, MUST be a JSON-LD context. SHOULD be exactly `"https://w3id.org/docmaps/context.jsonld"`  |
| `@graph` | MUST be a json-ld graph. |

By recommendation of the authors, this endpoint SHOULD NOT respond with elements of `"type": "docmap"`, though this is
allowed. It is generally recommended to respond with persistent objects like Actions, Assertions, and Manifestations.

A client MUST NOT assume that the sync is complete because the server did not obey the `limit` parameter. The only
semantics for a complete synchronization are a `202 Accepted` response, indicating that the sync is up-to-date and
may continue in the future, or a `204 No Content` or `410 Gone` response, indicating that the sync or session is no longer
useable and a new one should be started without the `from` parameter.

## Potential Impact

Aside from aforementioned benefits to having a shared protocol, there are risks and challenges to consider.

Existing implementations will likely be driven to adapt some of their choices to match the shared standard. If powerful
or large-audience stakeholders are unwilling to cooperate, competing standards are likely to emerge anyway. Additionally
there is risk that we commit as a group to solutions that scale poorly, are hard to understand, or present security risks.
See Security and Privacy Considerations below.

## Alternatives Considered

An alternative is considered where the API design of sync-forward and search APIs would be designed around DOIs
and Docmap IDs explicitly. In this alternative, a single Docmap is presumed likely to exist for a single DOI,
and the Docmap for that DOI may exist in several places and updated from time to time by the attester. This
useage is allowed in the Docmaps core specification, which is unopinionated about the handling of docmaps changing
over time.

However, given that a document or preprint may be argued for by various parties for various different reasons, it
is more appropriate to say that we have "a docmap for preprint P" rather than "the docmap for preprint P". It is
realistic to assume that a consumer of a docmap may intend to combine contents from two docmaps to make a new argument
for the document. Therefore for integrity and addressability reasons, the attester of the various integral pieces
of a docmap must already manage the pieces in their own right. Following this, we recommend in this document that
servers de-emphasize the ingestion of their docmaps for persistence purposes and serve the underlying elements in /synchronize.

Another considered solution to this same problem is to normalize the use of Docmaps that make direct reference to other
Docmaps rather than deconstructing them. In this flow, computation performed to combine two docmaps would treat each
sub-docmap as an Action in a Step in a super-Docmap. The Assertions present in a sub-Docmap would be exposed as assertions
in the containing Step. We do not pursue this solution because it confines computation performed across docmaps to Assertions
and not Steps -- if docmap D contains a step S and docmap E contains a step T, and S and T together are enough to warrant an
assertion but each alone is not, then a consumer of a docmap must handle a special case of recursion for all docmaps-as-steps.

## Security and Privacy Considerations

This proposal is primarily intended to describe Docmaps servers which may have secret signing keys but which do not
intend to keep any core Docmaps data secret. Therefore it is recommended to follow any organizational or regulatory
security requirements where they disagree with this proposal.

Systematically, the Docmaps ecosystem deals with the trustworthiness of scientific publications and other documents.
There is a security and integrity capability in the medium term that has not yet been explored, wherein a Public Key
Infrastructure (PKI) is incorporated with a Docmaps ecosystem to enable trustworthy computation to be performed on
Docmaps. This will be elaborated in future proposals, but it is of concern that this proposal is designed with the
future support of this feature set in mind.

This thinking is what motivated these choices in the proposal you have been reading:

- `peers` in `/info`: As currently designed, this is purely informational, but this struct can in future include
information about the trustworthiness of those peers.
- `/trust/\*` endpoints are reserved.
- `/publisher/{ID}` endpoint explicitly does not account for trust information,  and this RFC includes notes about
future additional endpoints that may enable trust information such as public keys under the reserved path `/trust`.

## References

- Sciety  server: https://sciety.org/docmaps/v1
- Sciety  source: https://github.com/sciety/api-prototype
- EMBO source: https://github.com/source-data/sd-graph
- TODO: link to extant biorxiv implementations
