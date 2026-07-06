# QubuiFS-CLI

Command-line tools and management-plane test utilities for QubuiFS.

## Contents

- `openapi/openapi.yaml`: bundled OpenAPI 3.0 contract for the QubuiFS management API.
- `openapi/paths/`: active path fragments from the QubuiFS management API source tree.
- `openapi/schemas/`: schema fragments used by the path definitions.
- `docs/design.md`: CLI design requirements, starting with management-plane testing.

## Current Scope

The first milestone focuses on management-plane connectivity checks and API tests driven by the OpenAPI contract. Filesystem data-path tooling is out of scope for the initial CLI.
