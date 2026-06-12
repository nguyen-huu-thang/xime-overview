<div align="center">

# 🧩 Xime Platform

### A microservice backend platform: your single entry point to every repository.

**English** · [Tiếng Việt](README-vn.md)

![Status](https://img.shields.io/badge/status-active%20development-orange)
![Stack](https://img.shields.io/badge/stack-Java%20·%20Python%20·%20Go-blue)
![Architecture](https://img.shields.io/badge/architecture-microservices-success)
![Frameworks](https://img.shields.io/badge/frameworks-Spring%20Boot%20·%20Xime-6E56CF)

</div>

---

**Xime Platform** is a backend foundation for building many products on top of one shared core. Its **Base Platform** provides the cross-cutting infrastructure every app needs - trust & security, authentication, user identity, data storage, notifications - so that applications (social, ecommerce, SaaS / AI…) reuse it instead of rebuilding it each time. The platform is distributed, multi-tenant and horizontally scalable.

It spans **many repositories**. Rather than scatter them, this page is the **single place to discover the whole project and jump to each repo**.

> 👇 **Looking for the code? Pick a repository below.**

---

## 📦 Repositories

### ⭐ Start here: the foundation

The platform is **polyglot**: its Java services run on **Spring Boot**, while its Python services run on **Xime Framework**, an in-house Python framework in the spirit of Spring Boot (automatic dependency injection, convention-based structure, code-first gRPC). Xime is built in-house, so it has its own repository:

| Repository | Description | |
| :--- | :--- | :--- |
| **[Xime Framework](https://github.com/nguyen-huu-thang/xime-framework)** | The in-house Python backend framework behind the platform's Python services | [![PyPI](https://img.shields.io/pypi/v/xime?label=pip%20install%20xime&color=3776AB&logo=pypi&logoColor=white)](https://pypi.org/project/xime/) |

### 🧱 Base Platform: shared infrastructure services

Reusable, business-agnostic services: written once, reused across every application.

| Repository | Description | Language |
| :--- | :--- | :---: |
| **[Trust Service](https://github.com/nguyen-huu-thang/trust-service)** | Trust infrastructure: internal CA, mTLS, JWT signing keys | `Java` |
| **[Identity Service](https://github.com/nguyen-huu-thang/identity-service)** | Authentication: JWT issuance, refresh tokens & rotation | `Java` |
| **[User Service](https://github.com/nguyen-huu-thang/user-service)** | Human Identity Domain: credentials, profile, account state | `Java` |
| **[Data Service](https://github.com/nguyen-huu-thang/data-service)** | Distributed object storage with capability-based permissions | `Python` |
| **[Notification Service](https://github.com/nguyen-huu-thang/notification-service)** | Outbound notifications (email first, SMS later): stateless | `Python` |

> 🛠️ Also planned: **Payment**, **Search** and **Analytics** services (not yet public).

---

## 🏗️ How it fits together

```text
┌──────────────────────────────────────────────────────────────┐
│  Application Layer   social · ecommerce · SaaS / AI  [planned] │
└──────────────────────────────────────────────────────────────┘
                          ▲  reuses
┌──────────────────────────────────────────────────────────────┐
│  Base Platform   trust · identity · user · data · notification │
└──────────────────────────────────────────────────────────────┘
                          ▲  built with
┌──────────────────────────────────────────────────────────────┐
│  Frameworks   Spring Boot (Java)  ·  Xime (Python)             │
└──────────────────────────────────────────────────────────────┘
```

**A backend foundation you build once and reuse across many products**: distributed-first, shared-nothing, horizontally scalable, multi-tenant and failure-tolerant.

Each service is written in the language that best fits its workload:

| Language | Best for | Services |
| :--- | :--- | :--- |
| **Java** (Spring Boot) | transaction-heavy, security-critical | trust · identity · user · payment |
| **Python** (Xime) | IO-bound, async, AI / data | data · notification · search · analytics |
| **Go** *(planned)* | realtime, streaming, video | live streaming · video calls |

---

<div align="center">

`distributed-first` · `shared-nothing` · `horizontally scalable` · `multi-tenant` · `failure-tolerant`

**Built by [@nguyen-huu-thang](https://github.com/nguyen-huu-thang)**

</div>
