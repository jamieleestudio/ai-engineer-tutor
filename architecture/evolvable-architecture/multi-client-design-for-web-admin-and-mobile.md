# Evolvable Architecture: Multi-Client Design for Web, Admin, and Mobile

## 1. Introduction

This document describes the approach to managing multi-client architectures, specifically when dealing with **web**, **admin**, and **mobile** client types. The goal is to **maintain clean separation of concerns** by structuring controllers based on the client type, and by ensuring that each client type is deployable independently without affecting the underlying business logic.

## 2. Architecture Overview

The system is built around the principle of **adapter-based architecture**, where:

- **Each client type (web, admin, mobile)** will have its own **Controller** layer.
- Business logic, represented by the **application** and **domain layers**, remains **shared** and independent of client type.
- **Boot modules** are used to control which client-specific controllers are loaded, allowing each client type to be deployed independently or together.

## 3. Project Structure

The project follows a **modular structure** where each module contains a distinct layer of the application. The structure supports flexible deployment and modular development.

### 3.1 Project Structure Example

```text
order
├─ boot
│  ├─ boot-monolith        # All modules and client types together (default for full deployment)
│  ├─ boot-web             # Web application deployment
│  ├─ boot-admin           # Admin panel deployment
│  └─ boot-mobile          # Mobile application deployment
├─ modules
│  ├─ order-api
│  ├─ order-application
│  ├─ order-domain
│  ├─ order-infrastructure
│  └─ order-interfaces     # Controllers are separated here
```
