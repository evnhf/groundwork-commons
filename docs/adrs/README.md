# Architecture Decision Records

This directory contains Architecture Decision Records (ADRs) for Groundwork Commons. ADRs document significant architectural and governance decisions made during the project.

## What is an ADR?

An ADR captures a single decision with its context, consequences, and rationale. ADRs help:

- Document the "why" behind technical and governance choices
- Provide historical context for future contributors
- Enable informed debate about alternatives
- Create accountability for decisions

## ADR Format

Each ADR follows a simple template:

- **Status**: Proposed, Accepted, Superseded, Deprecated
- **Context**: What circumstances led to this decision?
- **Decision**: What did we decide?
- **Consequences**: What are the implications (both positive and negative)?

See [template.md](template.md) for the full template.

## ADR Index

Currently, no ADRs have been created. As architectural and governance decisions are made, they will be documented here.

### Potential Future ADRs

Based on current musings, potential ADRs might include:

- Multi-node replication strategy
- Choice of distributed database (CouchDB, Scuttlebutt, Gun.js)
- Rationale for 3-5 node architecture
- Democratic governance model design
- Data sovereignty and privacy approach
- Consensus mechanisms for moderation
- Community fork process

## Creating a New ADR

1. Copy the [template.md](template.md) file
2. Name it `NNNN-short-title.md` (e.g., `0001-couchdb-replication.md`)
3. Fill in the sections with context, decision, and consequences
4. Set status to "Proposed"
5. After discussion/implementation, update status to "Accepted"
6. Add entry to this index with date and summary

## ADR Lifecycle

- **Proposed**: Decision is under consideration
- **Accepted**: Decision has been made and is being implemented
- **Superseded**: Decision has been replaced by a newer ADR (link to the new one)
- **Deprecated**: Decision is no longer relevant

## Related Documentation

- [Musings](../musings/README.md) - Exploratory thinking before decisions are made
- [Technical Documentation](../technical/README.md) - Implementation of decisions
- [Governance Framework](../governance/README.md) - Governance decisions in action
