# Fleet Infrastructure Knowledge & Configuration Platform

## Overview

This document describes the initial architectural concept for the BEAR_FLEETNET platform.

The project aims to create a centralized self-hosted environment for:

- fleet infrastructure inventory;
- configuration history tracking;
- monitoring;
- AI-assisted search and troubleshooting;
- technical knowledge accumulation.

---

# Main Components

## Baserow

Inventory platform and operational source of truth.

## PostgreSQL

Backend database.

## Gitea

Self-hosted Git platform for configuration history.

## Oxidized

Automated configuration backup collector.

## LibreNMS

Monitoring and SNMP discovery platform.

## AI Layer

RAG-based AI assistant over:

- configs;
- manuals;
- WhatsApp archives;
- diagrams;
- reports.

---

# Main Design Principles

- self-hosted first;
- incremental deployment;
- Git-centric configuration history;
- simple UX for onboard personnel;
- VPN-only management access;
- separation between inventory and monitoring.

---

# Initial Deployment Order

1. PostgreSQL + Baserow
2. Repository structure
3. Oxidized
4. LibreNMS
5. AI integration

---

# Target Deployment Host

```text
BearCore LP (BCL)
```

---

# Status

Initial concept phase.
