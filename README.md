# off-catalogue

The OFF Catalogue (OFC) is the repository and discovery service of the [Open Fitness Framework (OFF)](https://github.com/open-fitness-framework/off-specification) reference implementation. It is built on the shared library [off-core](https://github.com/open-fitness-framework/off-core).

The OFC:

- **stores pinned artifacts immutably:** signed fitness reports, fitness profiles, assertion definitions, methods, schemas and evidence files. Everything is content-addressed, and the OFC serves the exact bytes it received, by id and SHA-256 digest;
- **publishes profiles and definitions:** submissions are validated, every definition a profile pins must resolve, and versions are immutable;
- **ingests reports:** it checks their structure and verifies the signature against the issuer's published keys, to keep forged reports out. It never rewrites signed payloads;
- **makes evidence findable:** search by snapshot binding, definition, issuer and purpose, plus an issuer directory. Results are candidates and include negative reports.

The OFC is not a trust authority. It makes no claims about validity or fitness. Which issuers to trust is decided by each consumer's trust policy in the [off-evaluator](https://github.com/open-fitness-framework/off-evaluator).

The first version covers the OFF core: a REST API, public reading and writing for authenticated publishers.

**Status:** not yet implemented. See the [briefing](docs/briefing.md) for the planned scope.
