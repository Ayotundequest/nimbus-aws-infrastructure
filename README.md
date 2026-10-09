# Project Nimbus — AWS Cloud Infrastructure

Project Nimbus is the production-oriented AWS cloud infrastructure supporting **NimbusDesk**, an open-source support and service-request management application.

The project is being developed as an end-to-end Cloud Infrastructure Engineering environment covering network architecture, security, compute, data services, high availability, scalability, observability, recovery, cost engineering and operational troubleshooting.

## Architecture Approach

Infrastructure decisions follow:

    Requirement
        ↓
    Evidence & Assumptions
        ↓
    Constraints
        ↓
    Architecture Options
        ↓
    Trade-offs
        ↓
    Implementation
        ↓
    Validation
        ↓
    Measurement
        ↓
    Operational Learning
        ↓
    Refinement

The environment is developed incrementally rather than provisioning the complete target architecture before a workload exists.

## Current Architecture

### AWS Region

    Europe (London) — eu-west-2

### VPC

    nimbus-prod-vpc
    10.20.0.0/16

### Network Tiers

| Tier | Availability Zone A | Availability Zone B |
|---|---|---|
| Public | `10.20.1.0/24` | `10.20.2.0/24` |
| Application | `10.20.11.0/24` | `10.20.12.0/24` |
| Database | `10.20.21.0/24` | `10.20.22.0/24` |

The public subnets route internet-bound traffic through an Internet Gateway.

Application and database subnets currently retain VPC-local routing only. NAT Gateway provisioning has been intentionally deferred until a private application workload requires outbound internet connectivity.

## Current Project Status

### Completed

- AWS administrative access setup
- Dedicated production VPC
- Multi-AZ subnet architecture
- Public/application/database network segmentation
- Internet Gateway
- Public routing
- Explicit private application routing
- Explicit private database routing
- Network architecture validation

### Deferred by Design

- NAT Gateway provisioning

### Next

- Security and access foundation

## Documentation

Detailed network architecture:

`architecture/network-design.md`

Implementation and validation:

`docs/networking.md`

Operational incidents:

`incidents/`

Implementation evidence:

`evidence/`

## Engineering Principles

Project Nimbus prioritises:

- requirement-driven architecture;
- least privilege;
- private-by-default application and data tiers;
- multi-AZ resilience;
- replaceable compute;
- externalised persistent state;
- observability;
- evidence-based capacity planning;
- cost awareness;
- real troubleshooting evidence;
- documented architecture trade-offs.

Infrastructure abstractions such as Terraform, configuration management, containers and orchestration will be introduced during their corresponding engineering phases rather than used to bypass foundational AWS understanding.