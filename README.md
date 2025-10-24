# Groundwork Commons

A platform for neighborhood-scale social infrastructure

## About the Project

Most digital communication platforms treat physical proximity as irrelevant. They optimize for reach, engagement metrics, and network effects that scale to millions. This creates a gap: there's no good infrastructure for the 5-50 people who actually share a street or small neighborhood to coordinate about practical matters—lost pets, shared resources, local decisions, mutual aid.

**Groundwork Commons** is a platform for hyperlocal communities built on .NET with SQLite. A single primary node serves the community with continuous replication to backup locations (cloud storage, peer nodes, or local filesystems). When the primary becomes unavailable, communities can quickly restore from a replica and resume operations. The system uses proven, simple technology designed for the 5-50 person scale. Data stays within the neighborhood network, with no corporate intermediary required.

## Core Concepts

### Simple, Resilient Infrastructure
A single primary node serves the community, with continuous replication to backup locations. When the primary fails, another community member can restore from a replica and become the new primary. The simple architecture prioritizes reliability over complex distributed systems.

### Democratic Governance
Built-in proposal and voting system enables democratic decision-making. Members vote on governance changes, role assignments, and moderation decisions. Vote outcomes are automatically enforced by the system, preventing admin override.

### Community Ownership
Communities own their data and infrastructure. No corporate intermediaries, no centralized servers with access to your conversations. Node operators are community members, accountable through democratic processes.

### Hyperlocal Scale
Built for the 5-50 people who share physical proximity—a street, small neighborhood, or apartment building. The technology choices reflect this scale, favoring simplicity over features designed for millions of users.

## Key Features

- **Single-primary architecture** - One primary node with continuous replication to backups
- **Democratic governance** - Proposal and voting system with automated enforcement
- **Community ownership** - No corporate intermediaries, data stays local
- **Hyperlocal scale** - Optimized for neighborhoods of 5-50 people
- **Data sovereignty** - All data stays within the neighborhood network
- **.NET technology stack** - Built on proven, mature technology (ASP.NET Core, SQLite, Blazor)
- **Rich content** - Markdown formatting, image uploads, file attachments, link previews
- **Flexible storage** - Choose filesystem or cloud storage (S3-compatible) for media
- **Docker deployment** - Simple Docker Compose setup with web-based configuration wizard
- **Multiple interfaces** - Web application now, mobile apps in the future

## How It Works

1. **Deploy with Docker Compose** - Download docker-compose.yml and start containers with one command
2. **Setup Wizard** - Web-based wizard guides initial configuration (community name, admin account, storage)
3. **Primary Node** - One community member runs the primary node (cloud VPS or self-hosted)
4. **Continuous Replication** - Database and media continuously replicate to backup locations using Litestream
5. **Backup Locations** - Replicas stored in cloud storage, peer nodes, or local filesystems
6. **Manual Failover** - If primary fails, restore from replica and announce new primary URL
7. **Democratic Operations** - Members vote on governance changes through built-in proposal system

Communities start with a simple Docker deployment. The setup wizard handles initial configuration. The initial operator becomes the first admin. Over time, communities can elect additional admins and establish multiple backup locations for resilience.

## Project Status

**Current Phase**: Concept Development (October 2025)

We are in the early planning stages, defining architecture decisions and exploring technical approaches for distributed data replication and democratic governance mechanisms.

## Documentation

- [Documentation](docs/README.md) - Project documentation overview
- [Architecture Decision Records](docs/adrs/README.md) - Key technical and governance decisions that define the platform

## Code Repository

This repository contains the planning documentation and will eventually house the implementation.

## Getting Involved

This project is in early development. If you're interested in:

- **Contributing to design** - Review [Architecture Decision Records](docs/adrs/README.md) and provide feedback
- **Technical development** - Watch this repo for implementation opportunities
- **Starting a neighborhood** - Follow the project as deployment documentation develops

## License

This project is licensed under the [GNU General Public License v3.0](LICENSE).

The GPL v3 ensures that this infrastructure remains free and open. Communities have the freedom to run, study, modify, and distribute the software. Any modifications must also be released under GPL v3, preventing proprietary capture of community infrastructure.
