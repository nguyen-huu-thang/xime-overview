<div align="center">

<a href="https://xime.vn">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/xime-wordmark-white.png">
    <img src="assets/xime-wordmark.png" alt="Xime" width="240">
  </picture>
</a>

# Xime Platform

**Industry applications and business tools on a shared platform.**

**English** · [Tiếng Việt](README-vn.md)

![Status](https://img.shields.io/badge/platform-in%20development-0284c7?style=flat-square)
![Stack](https://img.shields.io/badge/stack-Java%20%7C%20Python%20%7C%20TypeScript-334155?style=flat-square)
[![PyPI](https://img.shields.io/pypi/v/xime?style=flat-square&label=Xime%20Framework&color=7c3aed)](https://pypi.org/project/xime/)

[**Visit Xime**](https://xime.vn) · [**Open ERP**](https://erp.xime.vn) · [**Explore the framework**](https://github.com/nguyen-huu-thang/xime-framework)

</div>

---

Xime brings together industry applications, reusable business modules and a shared backend platform. The same foundations support sign-in, accounts, organizations, usage plans, storage and notifications across products.

This repository is the public overview: a place to explore the applications, understand the architecture and find the code that is published openly.

## Explore Xime

| Website | What you will find |
| :--- | :--- |
| **[Xime Portal · xime.vn](https://xime.vn)** | The main website, product information and access to your Xime account, organizations and usage plans |
| **[Xime ERP · erp.xime.vn](https://erp.xime.vn)** | A shared workspace for HR and attendance, CRM, inventory, cashbook and tasks |

### Industry applications

| Management app | Focus | Sample website |
| :--- | :--- | :--- |
| **[Xime Dental](https://nhakhoa.xime.vn)** | Patients, appointments and clinic workflows | [Preview](https://mau-nhakhoa.xime.vn) |
| **[Xime Restaurant](https://nhahang.xime.vn)** | Tables, orders, kitchen and cashier workflows | [Preview](https://mau-nhahang.xime.vn) |
| **[Xime Rental](https://nhatro.xime.vn)** | Properties, rooms, contracts and rental charges | — |
| **[Xime Flowers & Gifts](https://shophoa.xime.vn)** | Products, custom orders and delivery workflows | [Preview](https://mau-shophoa.xime.vn) |

> **Trial phase.** Management apps use a Xime account. Sample websites illustrate customer-facing layouts; features are still being developed.

## One platform, three layers

| Layer | Responsibility |
| :--- | :--- |
| **Applications** | Industry workflows, public websites, management interfaces and the Portal |
| **Shared business modules** | HR and attendance, CRM, inventory, cashbook and tasks; reusable in industry applications and brought together in Xime ERP |
| **Base Platform** | Business-independent services for identity, organizations, usage plans, payments, storage, notifications and distributed infrastructure |

Organizations provide a common scope across products. Each application keeps its own business workflows, while shared capabilities are implemented once and reused.

### Shared infrastructure

| Area | Services and responsibilities |
| :--- | :--- |
| **Trust and sign-in** | Trust manages cryptographic trust; Identity handles authentication; User manages accounts and profiles |
| **Application identity** | Application and Agent manage application, bot and AI-agent identities |
| **Organizations** | Shared organization records, memberships, invitations and lifecycle |
| **Usage plans and payments** | Billing manages plans, subscriptions and entitlements; Payment handles payment processing and callbacks |
| **Storage and notifications** | Data provides object storage; Notification provides delivery and a persistent personal notification inbox |
| **Distributed infrastructure** | Placement provides shard health information; User Locator resolves login identifiers to user shards |

The architecture is designed for multiple tenants, independent services and horizontal growth. Current implementation details and operational documentation are maintained internally.

## Xime Framework

**One dependency graph, many entry points.**

[Xime Framework](https://github.com/nguyen-huu-thang/xime-framework) is the open-source Python framework used by the platform's Python services. It builds on FastAPI, SQLAlchemy and gRPC, adding constructor-based dependency injection, startup validation, explicit transactions and application lifecycle management.

It supports HTTP, WebSocket, gRPC, Unix sockets, MQTT, Modbus TCP and OPC UA, along with a multi-process runtime.

```bash
pip install xime
```

[Source code](https://github.com/nguyen-huu-thang/xime-framework) · [PyPI package](https://pypi.org/project/xime/) · [Getting started](https://github.com/nguyen-huu-thang/xime-framework/blob/main/docs/en/getting-started.md)

### Technology

| Technology | Role |
| :--- | :--- |
| **Java · Spring Boot** | Core platform services, authentication and transaction processing |
| **Python · Xime Framework** | Infrastructure services, shared business modules and industry backends |
| **TypeScript · Next.js · React** | Portal, ERP, management interfaces and public websites |
| **PostgreSQL** | Service-owned persistent data |
| **gRPC · mTLS** | Authenticated communication between backend services |

## Public source and historical repositories

**Xime Framework continues to be developed publicly.** The current platform services and applications are developed internally; their source code is not published in this repository.

The earlier public Base Platform repositories remain available as historical work. They were archived on **20 August 2026** and are read-only snapshots, rather than maintained releases of the current platform.

<details>
<summary><strong>Browse the seven archived Base Platform repositories</strong></summary>

| Historical repository | Area |
| :--- | :--- |
| [Trust Service](https://github.com/nguyen-huu-thang/trust-service) | Cryptographic trust and certificates |
| [Identity Service](https://github.com/nguyen-huu-thang/identity-service) | Authentication and tokens |
| [User Service](https://github.com/nguyen-huu-thang/user-service) | Accounts and profiles |
| [Application Service](https://github.com/nguyen-huu-thang/application-service) | Application identity |
| [Agent Service](https://github.com/nguyen-huu-thang/agent-service) | Bot and AI-agent identity |
| [Data Service](https://github.com/nguyen-huu-thang/data-service) | Object storage and permissions |
| [Notification Service](https://github.com/nguyen-huu-thang/notification-service) | Earlier notification delivery implementation |

</details>

---

<div align="center">

Built by [**Nguyễn Hữu Thắng**](https://github.com/nguyen-huu-thang)

[Xime Portal](https://xime.vn) · [Contact](https://xime.vn/lien-he) · [Tiếng Việt](README-vn.md)

</div>
