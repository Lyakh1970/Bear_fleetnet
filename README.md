# BEAR_FLEETNET

## Overview

BEAR_FLEETNET is a self-hosted fleet infrastructure knowledge and configuration management platform designed for fishing vessels and mixed marine IT/electronics environments.

The project combines:

- inventory management;
- network configuration history;
- monitoring;
- AI-assisted search;
- documentation indexing;
- collaborative workflows for shore engineering and onboard radio/electronics specialists.

The platform is intended for real operational marine environments:

- unstable WAN connectivity;
- Starlink / Iridium hybrid WAN;
- distributed vessels;
- limited onboard IT staff;
- heterogeneous marine equipment;
- centralized remote support.

---

# Planned Core Stack

## Inventory / Source of Truth

- Baserow
- PostgreSQL

## Configuration History

- Git / Gitea
- Oxidized

## Monitoring

- LibreNMS

## Connectivity

- Tailscale / ZeroTier / WireGuard

## AI Layer

- ChatGPT / Codex / RAG indexing

---

# Main Goals

- Centralized fleet visibility
- Automated configuration backups
- Standardized infrastructure documentation
- AI-assisted troubleshooting
- Reduced dependency on single engineering workstation
- Long-term knowledge accumulation

---

# Deployment Philosophy

The platform will be deployed incrementally.

No attempt should be made to implement the full stack at once.

Recommended order:

1. PostgreSQL + Baserow
2. Git repository structure
3. Oxidized
4. LibreNMS
5. AI integration

---

# Repository Structure

```text
ai.workflow/
 ├── tasks/
 ├── prompts/
 └── reports/

configs/

docs/

infra/

scripts/
```

---

# Current Status

Project bootstrap phase.
