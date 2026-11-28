# Briefing: OFF Catalogue (`off-catalogue`, OFC)

**Status:** draft for `/init` · **Specification:** [Open Fitness Framework (OFF) 0.3 draft](https://github.com/open-fitness-framework/off-specification/blob/main/spec/0.3/open-fitness-framework-spec.md) · **Builds on:** [`off-core`](https://github.com/open-fitness-framework/off-core)

## 1 Context

The Open Fitness Framework (OFF) defines fitness profiles, assertion definitions and signed fitness reports about immutable snapshots of data assets. Everything is referenced by `ArtifactRef`, an id plus a SHA-256 digest of the exact bytes (4.3). Evaluators need those bytes, and agents need to find which reports exist for a snapshot and which profiles exist for a purpose.

The reference implementation has three components on top of the shared library `off-core`:

| Component | Spec roles | Purpose |
| --- | --- | --- |
| OFF Engine (OFE) | Assessor, report issuer; snapshot binding | Runs fitness checks and issues signed reports. |
| **OFF Catalogue (OFC)**, this project | Repository, discovery; profile and definition publication | Makes reports, profiles, definitions and other artifacts findable and retrievable. |
| OFF Evaluator (OVA) | Evaluator, trust policy store, request building | Produces fitness decisions. |

The specification does not require a central registry (Section 12). The OFC is one place to publish and find artifacts, and it must not become a trust authority.

## 2 Users

- **Report issuers** (OFE deployments) that publish signed reports and their evidence.
- **Profile and definition publishers**, such as communities, standards teams and sponsors, that publish versioned profiles and assertion definitions.
- **Evaluators and agents** (OVA request builder, third-party agents) that look for reports about a snapshot and retrieve pinned artifacts by id and digest.
- **People** browsing which profiles, definitions and issuers exist.

## 3 Responsibilities and scope of the first version

The first version implements the OFF core and no extensions.

### 3.1 Immutable artifact store (Sections 4.3, 12)

- **One content-addressed store** holds every artifact type an `ArtifactRef` can point to: secured report bytes (JWS), profiles, assertion definitions, methods, schemas and schema bundles, codelists, configurations and evidence files.
- **Bytes are stored and served exactly as received.** The OFC never reformats or rewrites a payload under a signature (Section 12), and never pretty-prints JSON, because digests are byte digests (4.3).
- **Retrieval:**
  - by digest: `GET /artifacts/sha256/{digest}`;
  - by id + digest, as an `ArtifactRef` resolver for the OVA and agents.
  - A given id may have several digests (versions). A request with a digest that the OFC does not hold returns not found, never a different version.
- **Immutability:** artifacts are never changed. Withdrawal only hides an artifact from search; pinned retrieval keeps working, so that old decisions can be reproduced. Deletion is reserved for legal reasons.

### 3.2 Publication of profiles and definitions (Sections 5, 7.1)

- **Publishers submit** `FitnessProfile` and `AssertionDefinition` documents.
- **The OFC validates submissions with `off-core`:**
  - structure and field rules;
  - a profile's requirements are valid against the definitions they pin;
  - every definition a profile pins must be resolvable in the catalogue, or be submitted with it.
- **Versions are immutable.** A changed profile or definition must have a new version and digest (5.1). Resubmitting an existing id + version with different bytes is rejected.
- **Authoring happens elsewhere,** in an editor or a Git repository. The OFC only publishes.

### 3.3 Report ingest (Sections 6, 8.1, 12)

- **Publishers submit** secured reports (JWS compact serialisation) together with the evidence files they reference, or with references to evidence hosted elsewhere.
- **The OFC checks each report on ingest:**
  - parses it with `off-core` and checks the strict JWS and credential structure;
  - verifies the signature against the key with the matching `kid` in the issuer's published JWKS, which is fetched with the network controls of Section 13;
  - checks that every evidence item stored in the OFC matches its digest.
  
  This keeps garbage and spoofed reports out of the catalogue.
- **The ingest result is internal.** It is never exposed as validity or trust: the API has no "verified" or "valid" fields. Only an evaluator's own trust policy decides acceptance (Sections 8.2, 12).
- **Report ids:** a report whose issuer and id are already stored with different bytes is kept as well and flagged for inspection, because it is a possible equivocation (10.2). The OFC never decides which variant is right.

### 3.4 Discovery

- **Indexes, built from report payloads:**
  - asset, representation and snapshot ids and the snapshot digest (4.4);
  - issuer;
  - definitions used, with their parameters;
  - issue time, validity and assessment end time.
  
  These are the fields Section 12 suggests repositories expose.
- **Indexes for profiles and definitions:** purpose id, name, version and the definitions used.
- **Search API:**
  - which reports exist for this snapshot binding;
  - which reports use this definition;
  - which profiles serve this purpose;
  - which profiles use this definition;
  - list issuers.
  
  Results are candidates, not admitted evidence: the signed binding and the evaluator decide applicability (Section 12).
- **Issuer directory:** issuers are listed with a link to their JWKS, for findability only, never as trust.
- **DCAT:** an optional DCAT view describes discovery links (Section 12). It is a "should" for the first version.

### 3.5 Access control

- **Reading is public.**
- **Writing requires an authenticated publisher account**, with API tokens in the first version and OIDC later. Accounts control who may write; they never imply trust in the content. Publisher identity is recorded as provenance, separate from the report issuer.
- **Abuse controls:** size limits, rate limits and media-type checks.
- **Evidence access:** in the first version all stored evidence is public. Access-controlled evidence (E2 and Section 13) comes later, so publishers must not upload sensitive evidence.

### 3.6 Interfaces

- **REST API** with an OpenAPI description: artifacts, publication, ingest, search, issuers.
- **Minimal read-only web pages** for browsing profiles, definitions, issuers and the reports for a snapshot. This is a "should", not a must.
- **Delivery:** as a container image. Storage is a relational database for indexes and metadata plus an object store or file system for the bytes.

## 4 Out of scope for the first version

- Trust decisions, fitness decisions and any claim of validity or fitness (Section 12: unsigned catalogue claims are not OFF reports).
- Storing fitness decisions.
- Extensions: status-list hosting (E4) and access-controlled evidence (E2) come later.
- Authoring tools for profiles and definitions.
- Federation between catalogues, and harvesting from other sources.
- Assessing or storing the data assets themselves.

## 5 Constraints

- **Language:** Java, the same version and build tool as `off-core` (to be confirmed at `/init`).
- **Database and object store:** to be decided at `/init`. Recommendation: PostgreSQL and an S3-compatible store or local file system, with the hosting region in the EU.
- **Data protection:** publisher accounts contain personal data (name, e-mail address). Reports and evidence should not, but uploads are not inspected for it; the publisher terms say so.
- **Outgoing network** (JWKS retrieval) follows Section 13: allowed schemes and destinations, redirect, size and timeout limits.
- **Factory runtime:** the Software Factory image ships Node and Python only, so a `.factory/Dockerfile` with a JDK is needed.

## 6 Success criteria

- Every artifact is served byte-identical to what was submitted, and pinned retrieval with a wrong digest never returns content.
- The search for a snapshot binding returns all reports about it, including negative ones: no filtering by result.
- A forged report (signature not matching the issuer's JWKS) is rejected on ingest. A correctly signed report from an issuer the OFC has never seen is accepted.
- A profile whose definitions cannot be resolved is rejected at publication, and so is a changed profile without a new version.
- The OVA request builder can fetch everything it needs for the three published guides' scenarios from the OFC alone.

## 7 Candidate feature map

| # | Feature | Depends on |
| --- | --- | --- |
| 1 | Content-addressed artifact store and pinned retrieval | — |
| 2 | Publisher accounts and API tokens | — |
| 3 | Profile and definition publication with validation | 1, 2, off-core |
| 4 | Report ingest: structure, JWKS signature check, evidence digests, equivocation flag | 1, 2, off-core |
| 5 | Indexes and search API | 3, 4 |
| 6 | Issuer directory | 4 |
| 7 | Read-only web pages and DCAT view | 5, 6 |

## 8 Open questions for `/init`

- Database, object store and hosting.
- Whether ingest should also accept reports from issuers that publish no JWKS, for example signature unchecked but stored. The recommendation is no.
- License, as for the other OFF repositories.
