# Rhizome

Rhizome is a managed service provider (MSP) for Filecoin Onchain Cloud. Clients upload directly to a storage provider; Rhizome coordinates replication, verification, and repair across providers.

The service design is a work in progress. This repository does not yet contain a runnable service.

- [Direct tus upload proposal](docs/proposals/tus-upload-proposal.md) — standalone proposal for resumable uploads to storage providers.
- [Curio inclusion proof proposal](docs/proposals/curio-inclusion-proof-proposal.md) — PieceCID-based Merkle witnesses for staged and committed data.
- [Architecture](docs/architecture.md) — service boundaries and responsibilities.
- [Design index](docs/design/README.md) — current contracts, open work, and review gates.
- [Engineering principles](docs/engineering/principles.md) and [glossary](GLOSSARY.md) — implementation guidance and domain vocabulary.
