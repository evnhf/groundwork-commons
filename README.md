# Groundwork Commons

A distributed protocol for neighborhood-scale social infrastructure

## About the Project

Most digital communication platforms treat physical proximity as irrelevant. They optimize for reach, engagement metrics, and network effects that scale to millions. This creates a gap: there's no good infrastructure for the 5-50 people who actually share a street or small neighborhood to coordinate about practical matters—lost pets, shared resources, local decisions, mutual aid.

**Groundwork Commons** is a distributed platform for hyperlocal communities using multi-master replication across nodes operated by different neighbors. Each node runs an instance that continuously syncs with others, creating redundancy without requiring centralized infrastructure. The system uses distributed database technology for conflict-free replication, meaning data stays entirely within the neighborhood network. When one node goes offline, the others maintain service continuity. Member authentication happens locally, and there's no corporate intermediary with access to community data.

## Core Concepts

### Distributed Infrastructure
The platform can operate on a single node or scale to multiple nodes run by different community members. Multiple nodes continuously sync data with each other, providing technical resilience and preventing any single point of failure.

### Democratic Governance
Node operators are elected by members and can be removed through consensus voting. Moderation decisions require multi-node agreement, preventing any single administrator from unilateral control.

### Community Ownership
The governance model treats technical infrastructure as political infrastructure. Communities own their data and decision-making processes without corporate intermediaries.

### Hyperlocal Scale
Built for the 5-50 people who share physical proximity—a street, small neighborhood, or apartment building. The goal isn't to scale but to provide communities with owned infrastructure.

## Key Features

- **Distributed infrastructure** - Multi-node redundancy prevents single points of failure
- **Democratic governance** - Elected node operators with consensus-based decision making
- **Community ownership** - No corporate intermediaries, data stays local
- **Hyperlocal scale** - Optimized for neighborhoods of 5-50 people
- **Data sovereignty** - All data stays within the neighborhood network
- **Platform cooperativism** - Technical architecture aligned with cooperative governance

## How It Works

A neighborhood can start with a single node and add more nodes for resilience. When multiple nodes exist:

1. **Continuously sync data** using multi-master replication
2. **Provide redundancy** - if one node goes down, others maintain service
3. **Enable democratic governance** - consensus for moderation decisions (no single admin control)
4. **Stay local** - all data remains within the neighborhood network
5. **Support various hosting** - cloud VPS, self-hosted servers, or future mesh networks

Members connect to any available node. The underlying infrastructure is invisible to everyday users. A single-node setup provides the same features without the redundancy benefits.

## Project Status

**Current Phase**: Concept Development (October 2025)

We are in the early planning stages, defining architecture decisions and exploring technical approaches for distributed data replication and democratic governance mechanisms.

## Documentation

- [Architecture Decision Records](docs/adrs/README.md) - Key technical and governance decisions
- [Musings](docs/musings/README.md) - Exploratory thoughts and design explorations
- [Technical Documentation](docs/technical/README.md) - System architecture and implementation details
- [Governance Framework](docs/governance/README.md) - Democratic processes and node operator policies
- [Deployment Guide](docs/deployment/README.md) - Node setup and infrastructure guides

## Code Repository

This repository contains the planning documentation and will eventually house the implementation.

## Getting Involved

This project is in concept development. If you're interested in:

- **Contributing to design** - Review ADRs and musings, provide feedback
- **Technical development** - Watch this repo for implementation opportunities
- **Starting a neighborhood** - Follow deployment documentation as it develops

## License

This project is licensed under the [GNU General Public License v3.0](LICENSE).

The GPL v3 ensures that this infrastructure remains free and open. Communities have the freedom to run, study, modify, and distribute the software. Any modifications must also be released under GPL v3, preventing proprietary capture of community infrastructure.
