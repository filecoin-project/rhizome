# Working on Rhizome

- Start with the [architecture](docs/architecture.md), [design index](docs/design/README.md), and [glossary](GLOSSARY.md). Use **Rhizome** for the project and **MSP** for the provider role.
- Use the bundled [domain modeling skill](.agents/skills/domain-modeling/SKILL.md) when defining domain terms, editing the glossary, or recording architecture decisions.
- Keep each contract in its owning document under `docs/design/`. Link to that contract instead of copying it. Distinguish agreed direction, unresolved design, and behavior verified against an upstream dependency. Keep source citations pinned to the revision reviewed.
- Follow the [engineering principles](docs/engineering/principles.md). Keep domain policy in the owning module, use explicit SQL, and keep RPC, signing, and provider calls outside database transactions. Persist stable identities before external effects so interrupted work can be reconciled.
- Use one Go module for the service, with process wiring in `cmd/rhizome/` and service packages in `internal/`. Keep key custody outside the Rhizome process.
- Run the narrowest check that exercises a change. For documentation moves, verify relative links and section anchors. For workflow changes, verify recovery at the affected durable-write and external-effect boundaries.
