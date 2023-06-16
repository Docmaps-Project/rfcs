# Docmaps Server API Protocol

## Abstract

This proposal is for a standard API contract for servers which offer Docmaps to interoperate with each other. This includes
defining the endpoints that should be implemented for identifying the server, serving docmaps, searching for docmaps, including
rules such as what endpoint results must be stable/cacheable and which may be dynamic.

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

TODO: decide whether this is harmful to integrity efforts:
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

### Identifying the server

These endpoints are REQUIRED for all standard Docmaps servers. They enable consumers to make decisions about the identity
and trustworthiness of the server, and manage changes that may occur in the identity profile of the server.

#### GET /info
This endpoint provides information about the server to aid with future utilization:

TODO: should this be JSON-LD?

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

`**api_url**` : the URL with path prefix to this server. It SHOULD end with a slash.
It MUST conform to the endpoint URL constraints (see above).

`**api_version**` : MUST be exactly the version of this document (`"1.0.0"`).

`**ephemeral_document_expiry**` : MUST be an object that informs the user about conditions in which
ephemeral documents stop becoming available. If this server has conditions in which it expires these
documents, it MUST specify those here. If no guarantees are available and documents may expire immediately,
it SHOULD specify all expiry indicators as `0`. This property is applicable to search results, including Docmaps
for a given DOI, which are implicitly ephemeral due to the possibility that 

`**ephemeral_document_expiry.max_seconds**` : Optional. Clients should expect documents to expire after this duration in seconds.

`**ephemeral_document_expiry.max_retrievals**` : Optional. Clients should expect documents to expire after this number of retrievals.

`**peers**` : Optional field to refer clients to other Docmaps servers that may be of interest.
If present, MUST be an array of objects, and `peers[].api_url` MUST be a URL with path prefix
that directs to a server that conforms to this Protocol.
Clients MUST decide on their own whether to trust these other servers. Future updates to this
protocol will include how to specify trust information as part of this annotation.

### Querying a known object IRI

These endpoints are REQUIRED for all standard Docmaps servers. They enable consumers to retrieve objects in full which
the server can authoritatively or nonauthoritatively attest.

TODO: comment on ephemerality of docmap stuff.

#### GET /docmap/{ID}

MUST return a JSON body that is a decodable Docmap. MAY include a `@context` key
for users to interpret as a JSON-LD object. MUST NOT return a top-level `@graph`
key.

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

TODO: comment on future publisher integrity strategies.

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

TODO: comment on future publisher integrity strategies.

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

TODO: specifying handling of large sets of responses (pagination)?

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

TODO - what do i want to support here? Current utilization actually just does not work the way i recommend.

- since these are ephemeral endpoints, the ID for the docmap should never be the same as the query

#### GET /docmap_for/doi?subject={doi}
#### GET /docmap_for/iri?subject={iri}

### Batch queries for Indexing

These endpoints are OPTIONAL for Docmaps servers that wish to support being indexed. They allow a client to synchronize
its application state with the latest state of the server's dataset. This use-case has stricter requirements for data
integrity on the server's internal state management than the other use-cases.

#### GET /ldgraph

This endpoint exposes a sync-forward API for linked-data consumers to ingest wholesale the contents
of an RDF graph that stores docmaps. This is the preferred method of indexing.

**Request** query terms 

TODO fill out request body

- from: a sync head. must be number, can be timestamp or preferably a serial. if omitted, start from earliest known.
Conventionally, element(s) that match `from` are included.

**Response** MUST respond with an LD-graph  of objects that may be disjoint. These objects MUST all have
IRIs in their `ID` field.

TODO fill out body

- continue_from: what to include as `from` in the next request. Conventionally,  elements that match `from` are
not included.

#### GET /changes

This endpoint exposes a sync-forward API for linked or unlinked data that allows a client to confidently
model their  own datasets  based on  up-to-date information available from this server in a meterable
way.

If this endpoint exists, it MUST conform to this specification.

TODO: does this endpoint make any sense with the combination of /ldgraph? It would have to be "every
docmap we are capable of generating", or similar, but with the "what is a docmap?" questions, that
is not very well defined.

TODO: returns a paginated set of docmaps including URLs that the client can use to sync-forward.

## Potential Impact

Aside from aforementioned benefits to having a shared protocol, there are risks and challenges to consider.

Existing implementations will likely be driven to adapt some of their choices to match the shared standard. If powerful
or large-audience stakeholders are unwilling to cooperate, competing standards are likely to emerge anyway. Additionally
there is risk that we commit as a group to solutions that scale poorly, are hard to understand, or present security risks.
See Security and Privacy Considerations below.

**TODO:** reassess this section in light of the proposed contract.

## Alternatives Considered

- comment on making docmaps permanent objects

## Security and Privacy Considerations

[Identify any security or privacy concerns related to the proposed changes and describe how they will be addressed.]

**TODO:** reassess this section in light of the proposed contract.
**TODO:** assess forwards comaptibility with planned object integrity checking schemes.

## References

[List any relevant sources or documents that were used in developing this RFC.]

- TODO: link to extant Sciety implementations
- TODO: link to extant embo implementations
- TODO: link to extant biorxiv implementations
