# Design and Implementation of an IVOA Publishing Registry for the RSP

```{abstract}
The IVOA Registry is the distributed discovery layer that allows virtual observatory
services to be found by client tools such as PyVO and TOPCAT.

This technote describes the design and implementation of an OAI-PMH publishing
registry for the Rubin Science Platform (RSP), enabling RSP services such as TAP
and SIAv2 to be registered in the global IVOA Registry.
Rather than deploying a standalone registry application, the registry will be implemented
as a set of additional endpoints within Repertoire, the existing RSP service
and data discovery application.
```

## 1. Introduction

The IVOA Registry is a federated discovery system for Virtual Observatory resources, 
including services, datasets, and organisations.

Tools like PyVO and TOPCAT query searchable registries (GAVO, EuroVO) to find TAP and SIA
endpoints. If a service is not registered, it is not visible to users working
from those tools, and thus can only by used by manually entering it's URL.

The RSP does expose VOSI `/capabilities` and `/availability` endpoints on each
service which lets a client introspect a service once they know the URL, however that is not enough to make them discoverable.
To appear in VO registry searches, the RSP needs to publish its own resource
records through an OAI-PMH publishing registry and register that registry with
the IVOA Registry of Registries (RofR).

The RSP already runs a service and data discovery application in Repertoire.
Repertoire aggregates metadata about RSP services, datasets, and databases and
exposes them through a (JSON) discovery API at `/discovery`.

Rather than standing up a separate registry service, we can add OAI-PMH
endpoints directly to Repertoire and generate resource records from the
metadata it already holds.

### Goals

- Enable RSP IVOA services (TAP, SIAv2) to appear in global VO registry searches
- Implement an OAI-PMH publishing registry as part of Repertoire
- Model all resource records using `vo-models`
- Deploy the registry as an opt-in Phalanx feature, initially enabled on the IDFs
- Lay the groundwork for contributing Registry model extensions back upstream
  to `vo-models`

### Out of Scope

- Running a searchable (full-text) registry. We will publish records, but rely on
  existing global registries (GAVO, EuroVO) to index them, at least initially.
- Registering non-IVOA RSP services
- Scraping VOSI endpoints to auto-discover service URLs. Resource records will be
  assembled from Repertoire's existing discovery data

## 2. IVOA Registry Background

### 2.1 Registry Architecture

The IVOA Registry as a whole is a system that consists of three types of service:

| Node type | Role |
|-----------|------|
| **Publishing registry** | Holds authoritative resource records for a single authority and serves them via OAI-PMH |
| **Searchable registry** | Harvests records from many publishing registries and provides full-text search |
| **Registry of Registries (RofR)** | Holds the list of all publishing and searchable registries |

We (RSP) will need to run a publishing registry.
Once registered with the RofR, global searchable registries such as GAVO's
`dc.g-vo.org/tap` will harvest the RSP records automatically.

### 2.2 OAI-PMH

OAI-PMH (Open Archives Initiative Protocol for Metadata Harvesting) is the protocol used for registry harvesting.
A publishing registry handles a (small) fixed set of HTTP GET verbs which get used as query parameters:

| Verb | Purpose |
|------|---------|
| `Identify` | Returns repository name, admin contact, and protocol version |
| `ListMetadataFormats` | Returns the supported metadata schemas (for IVOA registries: `ivo_vor`) |
| `ListSets` | Returns the OAI set hierarchy (for IVOA: the `ivo_managed` set) |
| `ListIdentifiers` | Returns OAI identifiers (with optional date-range filter) |
| `ListRecords` | Returns full XML records (with optional date-range filter) |
| `GetRecord` | Returns a single record by identifier |

Every response, including error responses, is wrapped in a standard XML envelope:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<OAI-PMH xmlns="http://www.openarchives.org/OAI/2.0/"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://www.openarchives.org/OAI/2.0/
                             http://www.openarchives.org/OAI/2.0/OAI-PMH.xsd">
  <responseDate>2026-04-09T12:00:00Z</responseDate>
  <request verb="ListRecords" metadataPrefix="ivo_vor">
    https://data.lsst.cloud/registry/oai
  </request>
  <!-- verb-specific payload or <error> element -->
</OAI-PMH>
```

The `<request>` element echoes the request arguments as XML attributes. 
For `badVerb` and `badArgument` errors it must appear with no attributes at all
since the arguments are invalid.

The metadata format for IVOA registries is `ivo_vor`, whereas the payload of
each record is a VOResource XML doc.

### 2.3 Resource Record Types

Resource records in an IVOA publishing registry are defined by a family of
IVOA schemas:

| Schema | Record type | Use |
|--------|-------------|-----|
| VOResource 1.1 | `Resource`, `Organisation`, `Authority` | Organisations and authority identifiers |
| VODataService 1.2 | `DataService`, `CatalogService` | Data services with table metadata |
| TAPRegExt 1.0 | `TableAccess` | TAP service record |

The SIA v2 record type is yet to be confirmed. 
It may use `vs:DataService` directly with a `standardID="ivo://ivoa.net/std/SIA#query-2.0"` capability, rather than a SimpleDALRegExt-defined type.

At minimum the publishing registry must contain:

1. An **Authority** record claiming the IVO authority identifier
2. An **Organisation** record for Rubin Observatory
3. One **service record** per registered service

### 2.4 Registration Approaches

The IVOA wiki documents three ways to get services into the registry,
scaled by the number of resources:

- **Browser-based** (ESAVO interface): suitable for a handful of records that
  rarely change 
- **PURX proxy**: host static VOResource XML on a web server and point the PURX
  relay service at it (suitable for roughly a dozen records)
- **Publishing registry**: implement OAI-PMH independently and register directly
  with RofR. Recommended for 10 or more resources or frequent updates

The RSP already runs multiple IVOA services across multiple deployments and
we have plans to add more (ConeSearch, etc.), so a publishing registry is the right
choice. It also gives us full control over record content and when records
are updated.

## 3. Existing RSP Context

### 3.1 IVOA Services

The RSP currently deploys the following IVOA services:

**TAP** (`lsst-tap-service`, `tap-postgres`) provides ADQL query access to RSP catalog data.
Each TAP service exposes VOSI `/capabilities` and `/availability` endpoints.
We run several TAP instances, each serving different data releases.

**SIAv2** (`sia`) provides image discovery via SIA v2.
It exposes per-collection endpoints at `/api/sia/{collection}/query`, with VOSI
endpoints per collection.
The `sia` codebase already defines `BASE_RESOURCE_IDENTIFIER = "ivo://rubin/"`,
which as far as I am aware is the first and only RSP service that embeds an IVO identifier.

A **ConeSearch** service will also soon be available.
Registration of ConeSearch can be added later once the service is deployed.

### 3.2 Repertoire

Repertoire is an RSP (FastAPI) service that aggregates metadata about RSP services,
datasets, TAP schemas, and InfluxDB databases, exposing them through a JSON
discovery API at `/repertoire/discovery`.

The `RepertoireBuilder` client library produces a `Discovery` object.
At the top level it holds a `Services` object (with `internal` and `ui` maps)
and a `datasets` dict.
Each `Dataset` entry has its own `services` map of `DataService` objects, one
per data-access application for that dataset.
Each `DataService` carries a primary URL and a `versions` dict of `ApiVersion`
objects and each `ApiVersion` has a `url` and an optional `ivoa_standard_id`.
These two fields, access URL and IVOA standard ID are exactly what a
resource record needs for its capability block.

The publishing registry reads them directly from the discovery data rather than
re-specifying them, and supplements them with fields Repertoire has no
equivalent for: IVOIDs, display titles and admin contact.

## 4. Resource Records

### 4.1 IVO Authority Identifier

Every resource in the IVOA Registry is identified by an IVOID of the form:

```
ivo://<authority>/<path>
```

The authority segment is a globally unique string that an organisation claims by
registering an Authority record with a searchable registry.

IVOIDs are also OAI-PMH identifiers. Standard OAI-PMH uses `oai:<authority>:<local-part>`
identifiers, but IVOA registry convention is to use the IVOID directly (e.g. `ivo://rubin/tap`)
as the OAI identifier. 

The `sia` codebase already uses `ivo://rubin/` as its base resource identifier,
which is a reasonable starting point.

That said, the final authority string needs to be verified for global uniqueness
and confirmed before the registry goes into production. For the puspose of this document
`ivo://rubin` will be used as a preliminary choice.

Proposed IVOIDs:

| Record | IVOID |
|--------|-------|
| Authority | `ivo://rubin` |
| Organisation | `ivo://rubin/org` |
| TAP | `ivo://rubin/tap` |
| SIAv2 DP1 (IDF) | `ivo://rubin/sia/dp1` |

### 4.2 Authority Record

The Authority record claims ownership of the `ivo://rubin` namespace.
It must be the first record harvested from the registry and must carry a valid
admin email address.

### 4.3 Organisation Record

The Organisation record describes Rubin Observatory as an entity and needs to include
the observatory name, homepage URL, and contact information.

### 4.4 TAP Service Record

The RSP runs a number of TAP services (`tap`, `ssotap`, `livetap`, `consdbtap`, `ppdbtap`).
For the initial registry deployment we will have to discuss which of the TAP services we want being harvested. 

The TAP record is a `TableAccess` resource (TAPRegExt 1.0).

Key fields (IDF example):

| Field | Value |
|-------|-------|
| IVOID | `ivo://rubin/tap` |
| Standard ID | `ivo://ivoa.net/std/TAP` |
| Access URL | `https://data.lsst.cloud/api/tap` (derived from Repertoire) |
| Interface type | `ParamHTTP` |
| ADQL version | 2.1 |
| Upload support | yes |
| Table metadata | subset from TAP_SCHEMA |

Table metadata is optional for TAPRegExt compliance but is likely to improve discoverability
significantly. 

For the initial implementation we can leave it out and add it later once
we have a clearer strategy on how to do this, possible re-using the approach that we use to populate TAP_SCHEMA.

### 4.5 SIAv2 Service Record

The SIAv2 record is a `SimpleImageAccess` resource (SimpleDALRegExt 1.0), with
one record per data collection.

Key fields (IDF example):

| Field | Value |
|-------|-------|
| IVOID | `ivo://rubin/sia/dp1` |
| Standard ID | `ivo://ivoa.net/std/SIA#query-2.0` |
| Access URL | `https://data.lsst.cloud/api/sia/dp1/query` (derived from Repertoire) |
| Interface type | `ParamHTTP` |

### 4.6 Required VOResource Fields

Every resource record must satisfy the following VOResource 1.1 requirements,
regardless of the specific record type.

**Root element attributes**

| Attribute | Requirement |
|-----------|-------------|
| `status` | Must be `active`, `inactive`, or `deleted`. All our records will use `active`. |
| `created` | ISO 8601 datestamp when the record was first created. Set once and never changed. |
| `updated` | ISO 8601 datestamp of the last meaningful change. Bumped manually in Phalanx values when a record changes. |

**`<curation>` block** (required)

Must contain at minimum:
- `<publisher>` - the name of the publishing organisation (e.g. `Rubin Observatory`)
- `<contact>` with a `<email>` sub-element - used by harvesters to report problems

**`<content>` block** (required)

Must contain at minimum:
- `<subject>` - one or more subject keywords (e.g. `Astronomy`, `Catalogs`)
- `<description>` - Description of the resource
- `<referenceURL>` - a URL for information about the resource

**Capability block**

Each `<capability>` element carries a `standardID` attribute (the IVOA standard
identifier, e.g. `ivo://ivoa.net/std/TAP`). 
The nested `<interface>` element must have an `xsi:type` attribute set to a schema-qualified type such as
`vs:ParamHTTP` and the `<accessURL>` inside it must carry `use="full"`.

These constraints need to be met as they will be validated by registry validators before submission to the RofR.

## 5. vo-models

### 5.1 Overview

`vo-models` (https://github.com/spacetelescope/vo-models) is a Python library
from the Space Telescope Science Institute that provides Pydantic/XML models
for IVOA schemas. It is already a dependency of certain RSP applications such as `sia` (used for VOSI models).

The library covers a number of schemas needed for the publishing registry:

| Module | Standard | Used for |
|--------|----------|---------|
| `vo_models.voresource` | VOResource 1.1 | `Resource`, `Organisation`, `Authority` base types |
| `vo_models.vodataservice` | VODataService 1.2 | `DataService`, table/column metadata |
| `vo_models.tapregext` | TAPRegExt 1.0 | `TableAccess` TAP service record |
| `vo_models.vosi` | VOSI 1.1 | Capabilities parsing |
| `vo_models.voregistry` | VORegistry 1.1 | Registry-level types |

Models are built on `pydantic-xml`, so they validate like normal Pydantic objects
and serialise to IVOA compliant XML via `.to_xml()`.

### 5.2 Example: Generating a TAP Resource Record

The following snippet shows how the `ResourceRecordFactory` might build a TAP
record. The access URL and standard ID come from Repertoire's discovery data, whereas
the IVOID, title  and datestamp come from the registry config in Repertoire.

```python
from vo_models.voresource.models import Capability, Interface, AccessURL
from vo_models.tapregext.models import TableAccess

tap_url = discovery.datasets["dp02"].services["tap"].versions["v1"].url
standard_id = discovery.datasets["dp02"].services["tap"].versions["v1"].ivoa_standard_id

tap_record = TableAccess(
    created="2026-04-09T00:00:00Z",
    updated=registry_config.services["tap"].updated,
    status="active",
    title=registry_config.services["tap"].title,
    identifier=registry_config.services["tap"].ivoid,
    curation=...,
    content=...,
    capability=[
        Capability(
            standard_id=standard_id,
            interface=[
                Interface(
                    type="ParamHTTP",
                    role="std",
                    access_url=[AccessURL(use="full", value=str(tap_url))],
                )
            ],
        )
    ],
)

xml_bytes = tap_record.to_xml(encoding="unicode")
```

This will generally be much cleaner than having to manage raw XML or Jinja template, 
and gives us schema-level correctness checks at build time rather than at harvest time.

### 5.3 Upstream Contribution

`vo-models` does not currently cover all schemas needed for a publishing registry.
One gap for example is: 
- **OAI-PMH response envelope types** - the `<OAI-PMH>`, `<request>`, `<header>`, and `<record>`
  wrapper elements that every response must include. Without these models the registry cannot
  produce a valid XML response.

This will have to be implemented within Repertoire first using the same `pydantic-xml` approach,
then contributed upstream once it has been validated in production.

## 6. Architecture

```{figure} diagram.png
:figclass: technote-wide-content
:scale: 50%

Proposed Architecture Diagram
```

### 6.1 Registry as Part of Repertoire

The OAI-PMH registry will be a new router added to the existing Repertoire FastAPI
application. 
There are obvious benefits to this, as this way there is no separate service to deploy
and maintain, and the registry reuses Repertoire's configuration loading, startup
lifecycle and Phalanx values structure, avoiding duplicated configuration.

The registry router will be mounted at `/registry` and gated on a
`registry.enabled` config flag, so it can be turned on for IDF environments and left off
everywhere else where it is not needed.

### 6.2 Resource Record Strategy

Records will be built at application startup by a `ResourceRecordFactory` that
will pull from:

- **Repertoire's discovery data** - the `Discovery` object produced by
  `RepertoireBuilder` already contains the access URL (`ApiVersion.url`) and
  IVOA standard ID (`ApiVersion.ivoa_standard_id`) for each registered service
  version. The access URL is specific to each environment and generated via
  Repertoire from `base_hostname`, so reading it from `Discovery` avoids any 
  duplication. The `ivoa_standard_id` is also present in the Phalanx rules,
  so we will also use it from there. 
  Note that `ivoa_standard_id` is only present if it is set in the Phalanx
  `values.yaml` rule for that application. Including it for the `tap` and `sia`
  rules is a prerequisite.

- **Registry-specific config** - fields that Repertoire has no equivalent for
  (IVOIDs, admin contact, datestamps, organisation homepage)
  come from the `registry:` config block described in Section 9.

The factory will look up each service entry by Phalanx app name (and
dataset name for per-collection services like SIA), fetch the URL and standard
ID from the discovery data and constructs a `vo-models` record object.
Records will be cached in a `RecordStore` at startup and served from memory on
every OAI-PMH request.

Service URLs have a single source of truth, which is the Repertoire's rule templates.
This way any URL changes flow through to published records automatically without touching
the registry config.

### 6.3 Record Datestamps

Worth noting that there is a `datestamp` on each record that the OAI-PMH harvesters use to 
determine what has changed since their last pull. 
For each service record, this datestamp is the `updated` field in the registry config. 

Thus for our RSP Registry we will have to bump it in the Phalanx values whenever the content of 
a resource record changes meaningfully (e.g. title change, new collection, changed URL). 

The initial implementation will not track deleted records, records can only
be added or updated. If we for whatever reason want support for soft-deleting in future, we can add it later.

### 6.4 Component Overview

```
Repertoire FastAPI app
|-- /discovery                  (existing)  JSON discovery API
|-- /registry                   (new)
|   |-- OAI-PMH handler         Processes verb, metadataPrefix, from/until params
|   |-- RecordStore             Holds VOResource objects keyed by IVOID
|   |-- ResourceRecordFactory   Builds records from registry config + discovery data
|   |-- OAI-PMH serialiser      Wraps records in OAI-PMH XML envelope
```

The `RecordStore` will be populated at application startup by the
`ResourceRecordFactory`, which combines the registry config and Repertoire's
discovery data.

The OAI-PMH handler queries this store and serialises responses.

### 6.5 Startup Validation

The registry config is keyed by Phalanx application name and dataset name, which
the factory uses to look up paths in the `Discovery` object. This mapping however is
implicit, and there is nothing in the config structure that enforces that a key like
`tap` or `sia/dp1` resolves to a valid entry in `Discovery`.

To avoid serving malformed records silently, the `ResourceRecordFactory` must
validate at startup:

- Every service entry in the registry config resolves to a matching path in `Discovery`
- `ivoa_standard_id` is present on every resolved `ApiVersion`. A missing standard ID
  would produce a `<capability>` with no `standardID` attribute, leading to a failed registry validation.
- All required VOResource fields (see Section 4.6) are present and non-empty

If any of these checks fail, Repertoire should refuse to start rather than serve
an incomplete or invalid registry. A broken publishing registry would probably be worse than no
registry at all  since harvesters may ingest and propagate the invalid records.

## 7. API

All OAI-PMH requests go to a single endpoint:

```
GET /registry/oai
```

The verb is a query parameter (`?verb=ListRecords&...`).
Responses are always `Content-Type: text/xml`.

### 7.0 Error Responses

OAI-PMH requires that invalid or unsatisfiable requests return a well-formed
XML error response, not an HTTP error code.
The error codes that must be implemented are:

| Code | Trigger |
|------|---------|
| `badVerb` | Unknown or missing `verb` parameter |
| `badArgument` | Missing required argument or unknown argument for the verb |
| `cannotDisseminateFormat` | Unsupported `metadataPrefix` (anything other than `ivo_vor`) |
| `idDoesNotExist` | IVOID passed to `GetRecord` not found in the `RecordStore` |
| `badResumptionToken` | Resumption token is invalid or expired (not used in the initial implementation) |
| `noRecordsMatch` | Date filter on `ListIdentifiers`/`ListRecords` matches no records |
| `noSetHierarchy` | `ListSets` called but no sets defined (not applicable here - we define `ivo_managed`) |
| `noMetadataFormats` | `GetRecord` called for an identifier that exists but has no associated metadata formats |

All error responses return HTTP 200 with an `<error>` element in the OAI-PMH
envelope.

### 7.1 Identify

```
GET /registry/oai?verb=Identify
```

Returns:
- Repository name
- Base URL
- Protocol version (2.0)
- Admin email,
- Earliest datestamp
- Deleted record policy
- Granularity.

```xml
<OAI-PMH xmlns="http://www.openarchives.org/OAI/2.0/"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://www.openarchives.org/OAI/2.0/
                             http://www.openarchives.org/OAI/2.0/OAI-PMH.xsd">
  <Identify>
    <repositoryName>Rubin Observatory VO Publishing Registry</repositoryName>
    <baseURL>https://data.lsst.cloud/registry/oai</baseURL>
    <protocolVersion>2.0</protocolVersion>
    <adminEmail>our-email</adminEmail>
    <earliestDatestamp>2026-04-09T00:00:00Z</earliestDatestamp>
    <deletedRecord>no</deletedRecord>
    <granularity>YYYY-MM-DDThh:mm:ssZ</granularity>
  </Identify>
</OAI-PMH>
```

### 7.2 ListMetadataFormats

```
GET /registry/oai?verb=ListMetadataFormats
```

Returns the single supported metadata prefix `ivo_vor`, pointing to the
VOResource 1.1 schema.

### 7.3 ListSets

```
GET /registry/oai?verb=ListSets
```

Returns the `ivo_managed` set, which IVOA harvesters use to select only IVOA
resource records.

### 7.4 ListIdentifiers

```
GET /registry/oai?verb=ListIdentifiers&metadataPrefix=ivo_vor
GET /registry/oai?verb=ListIdentifiers&metadataPrefix=ivo_vor&from=2026-04-09T00:00:00Z
```

Returns OAI headers (identifier + datestamp) for all records, with optional
date-range filtering. 

Every record header must include `<setSpec>ivo_managed</setSpec>`, which is how IVOA harvesters select only IVOA 
resource records when harvesting a registry.

This for example would at minimum list `ivo://rubin`, `ivo://rubin/org`, `ivo://rubin/tap`, `ivo://rubin/sia/dp1` in our case.

### 7.5 ListRecords

```
GET /registry/oai?verb=ListRecords&metadataPrefix=ivo_vor
```

Returns full VOResource XML for all records, with the same `from`/`until`
date filter as `ListIdentifiers`. The initial record count is small enough
that resumption tokens are probably not needed.

### 7.6 GetRecord

```
GET /registry/oai?verb=GetRecord&metadataPrefix=ivo_vor&identifier=ivo://rubin/tap
```

Returns a single VOResource record by IVOID.

## 8. Design Decisions

### 8.1 Why Not the NOIRLab Registry

NSF NOIRLab has developed an open-source OAI-PMH publishing registry at
https://gitlab.com/nsf-noirlab/csdc/vo-services/noirlab-vo-registry
This is a working Python implementation with Terraform IaC for GCP and active development.

It is a reasonable solution in isolation but a poor fit for us as we have a well established
pattern for how our applications are constructed deployed, managed, with a well known stack.

Even extracting components would be likely more complicated than writing from scratch due to the 
significant re-writing that would have to take place for it to fit within our FastAPI/Pydantic v2/Safir/vo models
stack.

This is still a useful reference for OAI-PMH behaviour and VOResource
record structure, but building within Repertoire is most likely the right call.

### 8.2 Resource Record Assembly

The simplest alternative would be to store fully-formed VOResource XML as
static files in the Phalanx values. While this would work, service URLs would then
exist in two places,  Repertoire's rule templates and the registry XML. 

This would lead to potential diverging, which is an operational hazard that we want
to avoid.

Deriving access URLs and standard IDs from Repertoire's discovery
data, avoids any mismatch between what Repertoire advertises and what the
registry publishes. 

The registry config carries only any IVOA-specific fields that the existing Repertoire discovery configuration
has no concept of.

### 8.3 Registry Deployment: Inside Repertoire vs Standalone Service

The registry could be deployed in two ways.

**Option A - inside Repertoire**: As discussed above, the OAI-PMH endpoints are added as a
conditional router to the existing Repertoire FastAPI application, gated on
`registry.enabled`. 
The `ResourceRecordFactory` reads URLs and standard IDs
directly from Repertoire's in-memory `Discovery` object at startup. 

No new Phalanx application is needed and the existing
configuration, logging, and deployment infrastructure are reused.

**Option B - standalone service**: Another option worth considering, is that the registry 
runs as its own Phalanx application. 

It calls Repertoire's public `/discovery` JSON endpoint at startup as a client
to retrieve service metadata. 

The service has an independent deployment lifecycle and can be restarted, scaled, or 
changed without touching Repertoire.

The main advantage of Option B is decoupling. Repertoire is an actively
developed service and is a core part of the RSP.
New features, schema changes and dependency updates mean it is likely to be redeployed frequently. 
For a published OAI-PMH endpoint registered with the IVOA RofR,
frequent downtime is undesirable, and we need to avoid any downtime which would affect external harvesting.
A standalone registry can be kept deliberately boring and stable regardless of what happens to Repertoire.


The main advantage of Option A is simplicity. While adding a new application is a well-established and simple process, 
it is still an additional service to deploy, monitor, and maintain. 

For an initial implementation with a handful of records and low harvest frequency, the operational overhead of Option B 
seems hard to justify.

Thus for the first iteration of the registry we will go with Option A and implement
it inside Repertoire. The decoupling concerns are real but manageable, and if we find
stability issues or feel the registry warrants its own lifecycle, extracting it into a
standalone service is a well-defined follow-on.

The availability concern is mitigated in practice by Kubernetes rolling updates.
Repertoire pods should in practice be replaced one at a time with zero-downtime deploys during upgrades, so the
OAI-PMH endpoint remains reachable across normal redeployments. 

The only risk is then an unplanned crash or a startup failure, which would take the endpoint down
briefly until Kubernetes restarts the pod. 
This is acceptable for the initial deployment given what we expect to be low harvest frequency of IVOA harvesters.


### 8.4 Per-Environment Registry

Each RSP deployment has its own service URLs and potentially a different set of
active services. Our plan is to deploy the registry for our integration and development IDF environments by 
setting the `registry.enabled: true`, but to only register prod in the RofR.
This way we can still exercise and test our Registry service, and potentially allow other tools to use it internally in the 
RSP (Firefly) for service discovery.

## 9. Configuration

The registry is controlled by a new `registry` section in Repertoire's
configuration schema.

Repertoire already supplies most of what a resource record needs:

- **Service access URLs** are computed by `RepertoireBuilder` from Jinja
  templates and `base_hostname`, so they are not re-specified here
- **IVOA standard IDs** come from `ApiVersion.ivoa_standard_id` in the
  discovery model (e.g. `"ivo://ivoa.net/std/SIA#query-2.0"`)
- **Dataset descriptions** come from `DatasetConfig.description`
- **The OAI-PMH base URL** is derived from `base_hostname` as
  `https://{base_hostname}/registry/oai`

The registry config therefore contains only what Repertoire cannot supply: IVO
identifiers, display titles, admin contact and datestamps. 

Service entries are keyed by Phalanx application name (and dataset name for per-collection
services like SIA) so the factory can look up URLs and standard IDs from the
discovery data without duplication.

Each record carries two datestamp fields:

- `created` - the date the record was first published. Set once and never changed,
  even when the record content is updated. This is distinct from `updated`.
- `updated` - the date of the last meaningful change to the record. This is the
  field OAI-PMH harvesters use to detect changes. It must be bumped in the Phalanx
  values whenever the record content changes (see Section 6.3).

```yaml
registry:
  enabled: true
  authority: "ivo://rubin"
  repositoryName: "Rubin Observatory VO Publishing Registry"
  adminEmail: "our-email"

  organisation:
    ivoid: "ivo://rubin/org"
    title: "NSF-DOE Vera C. Rubin Observatory"
    homepage: "https://rubinobservatory.org"
    created: "2026-01-01T00:00:00Z"
    updated: "2026-04-09T00:00:00Z"

  services:
    tap:
      ivoid: "ivo://rubin/tap"
      title: "Rubin Observatory TAP Service"
      created: "2026-01-01T00:00:00Z"
      updated: "2026-04-09T00:00:00Z"

    # For per-collection services, the second key is the dataset name,
    # matching config.available_datasets.
    sia:
      dp1:
        ivoid: "ivo://rubin/sia/dp1"
        title: "Rubin Observatory SIAv2 Service (DP1)"
        created: "2026-01-01T00:00:00Z"
        updated: "2026-04-09T00:00:00Z"
```

When `registry.enabled` is `false` (the default), the `/registry` router is
not mounted.

The Authority record is generated from the top-level `authority`,
`repositoryName`, and `adminEmail` fields, and thus no separate Authority block is
needed in the config.

### 9.1 IVO Authority Confirmation

As discussed earlier, before going into production, we need to verify `ivo://rubin` as unique and
formally claim it by submitting an Authority record to a searchable registry.

The procedure is described at
https://wiki.ivoa.net/twiki/bin/view/IVOA/GettingIntoTheRegistry.

## 10. Phalanx Deployment

The registry will ship as part of the existing `repertoire` Phalanx application
at `applications/repertoire/`.

`registry.enabled` is set per environment in `values-{env}.yaml`:

```yaml
repertoire:
  config:
    registry:
      enabled: true
      authority: "ivo://rubin"
      repositoryName: "Rubin Observatory VO Publishing Registry"
      adminEmail: "our-email"
      ...
```

Once the registry is live on the IDF, the OAI-PMH endpoint is run through tn RofR validator and then submitted to the Registry of Registries.

After that, our current understanding is that GAVO, EuroVO and other global searchable registries will harvest
RSP records automatically and that changes typically should propagate within a day.

## 11. Testing and Validation

### 11.1 Unit Tests

The OAI-PMH handler must have unit tests covering all six verbs and all eight
error codes.

### 11.2 Schema Validation

Generated VOResource XML should be validated against the IVOA schemas as part of
the test suite. `vo-models` provides some of this via Pydantic validation at
construction time, but serialised XML should also be validated against the
published XSD schemas to catch namespace or attribute errors.

### 11.3 IVOA Validator

Before submitting to the RofR, the live OAI-PMH endpoint must be validated by at least one of the IVOA registry validator. 
This checks that the endpoint returns valid OAI-PMH responses and that the VOResource records are schema-compliant. 
The validator URL is linked from the IVOA "Getting Into the Registry" page.

This should be part of the deployment checklist for the first IDF deployment.

### 11.4 Integration Environment

As described in Section 8.4, the registry will be enabled on the integration IDF
environment before production. This gives an opportunity to run the IVOA validator
against a live endpoint and verify end-to-end harvesting behaviour before
registering the production URL with the RofR.

## References

- IVOA Registry: Getting Into the Registry - https://wiki.ivoa.net/twiki/bin/view/IVOA/GettingIntoTheRegistry
- VOResource 1.1 - https://www.ivoa.net/documents/VOResource/
- VODataService 1.2 - https://www.ivoa.net/documents/VODataService/
- TAPRegExt 1.0 - https://www.ivoa.net/documents/TAPRegExt/
- SimpleDALRegExt 1.0 - https://www.ivoa.net/documents/SimpleDALRegExt/
- OAI-PMH 2.0 - https://www.openarchives.org/OAI/openarchivesprotocol.html
- vo-models - https://github.com/spacetelescope/vo-models
- NOIRLab VO Registry - https://gitlab.com/nsf-noirlab/csdc/vo-services/noirlab-vo-registry
- Repertoire - https://github.com/lsst-sqre/repertoire
- SIA service - https://github.com/lsst-sqre/sia
- Safir - https://safir.lsst.io/
- Phalanx - https://phalanx.lsst.io/
- PyVO - https://pyvo.readthedocs.io/
