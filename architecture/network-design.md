# Project Nimbus — AWS Network Architecture

## Overview

Project Nimbus is the AWS cloud infrastructure supporting NimbusDesk, an open-source support and service-request management application.

This document defines the initial network architecture for the NimbusDesk production environment.

The network was designed before application deployment because the known workload requirements were sufficient to establish the required network boundaries. Capacity-sensitive decisions such as EC2 sizing, Auto Scaling thresholds and database sizing remain subject to workload measurement and load testing.

---

## Design Requirements

The network must:

- provide a dedicated production VPC;
- support deployment across multiple Availability Zones;
- separate public, application and database tiers;
- prevent application servers from requiring direct public exposure;
- prevent database resources from requiring public exposure;
- support an internet-facing load-balancing tier;
- support controlled outbound connectivity from private application resources when required;
- provide sufficient address space for future growth;
- support future EC2, ALB, Auto Scaling and RDS deployment.

---

## Region

The initial environment is deployed in:

    Europe (London) — eu-west-2

The initial NimbusDesk deployment is UK-oriented. Region selection may be reassessed if future workload requirements introduce different latency, regulatory, service-availability or geographic requirements.

---

## VPC

    Name: nimbus-prod-vpc
    CIDR: 10.20.0.0/16

The /16 address space provides sufficient room to divide the environment into smaller network tiers while retaining address capacity for future expansion.

---

## Subnet Architecture

The VPC is divided across two Availability Zones.

| Tier | Availability Zone A | Availability Zone B |
|---|---|---|
| Public | 10.20.1.0/24 | 10.20.2.0/24 |
| Application | 10.20.11.0/24 | 10.20.12.0/24 |
| Database | 10.20.21.0/24 | 10.20.22.0/24 |

The implemented subnets are:

| Subnet | CIDR | Availability Zone | Purpose |
|---|---|---|---|
| nimbus-public-a | 10.20.1.0/24 | eu-west-2a | Public infrastructure |
| nimbus-public-b | 10.20.2.0/24 | eu-west-2b | Public infrastructure |
| nimbus-app-a | 10.20.11.0/24 | eu-west-2a | Private application workloads |
| nimbus-app-b | 10.20.12.0/24 | eu-west-2b | Private application workloads |
| nimbus-db-a | 10.20.21.0/24 | eu-west-2a | Private database resources |
| nimbus-db-b | 10.20.22.0/24 | eu-west-2b | Private database resources |

The numbering convention makes the tier and Availability Zone identifiable from the address plan.

---

## Internet Connectivity

An Internet Gateway named:

    nimbus-prod-igw

is attached to the Nimbus production VPC.

The public route table contains:

    10.20.0.0/16 → local
    0.0.0.0/0    → nimbus-prod-igw

The following subnets are explicitly associated with the public route table:

    nimbus-public-a
    nimbus-public-b

The presence of an Internet Gateway alone does not make a subnet public. The public classification results from the subnet's route-table association and its default route toward the Internet Gateway.

Individual resources still require the appropriate addressing and security configuration before direct internet communication is possible.

---

## Application Routing

Separate route tables are maintained for each application subnet:

    nimbus-app-a-rt
    nimbus-app-b-rt

During the initial network implementation they contain only the VPC local route:

    10.20.0.0/16 → local

Therefore the application subnets can communicate with destinations routed within the VPC but currently have no general internet route.

The target production architecture will provide controlled outbound connectivity through an Availability-Zone-local NAT Gateway:

    App-A → NAT-A
    App-B → NAT-B

This prevents the intended resilient architecture from making one application's outbound path dependent on a NAT Gateway located in the other Availability Zone.

---

## Database Routing

Separate route tables are maintained for the database subnets:

    nimbus-db-a-rt
    nimbus-db-b-rt

They currently contain:

    10.20.0.0/16 → local

No default internet route has been added.

Database internet connectivity will not be introduced without an explicit workload requirement.

---

## NAT Gateway Decision

### Target Production Architecture

The target resilient architecture uses one NAT Gateway for each active application Availability Zone.

### Current Lab Implementation

NAT Gateways have intentionally not yet been provisioned.

### Reason

No application instances currently require outbound internet connectivity, while NAT Gateways introduce ongoing infrastructure cost.

Provisioning resources before a workload exists would add cost without allowing meaningful connectivity validation.

### Trigger for Reassessment

NAT connectivity will be implemented when the private application workload is deployed and requires outbound access.

This is a lab implementation decision and does not change the target production architecture.

---

## Current Traffic Paths

### Public Internet Path

    Public subnet
        ↓
    Public route table
        ↓
    0.0.0.0/0
        ↓
    Internet Gateway
        ↓
    Internet

### Application-to-Database Routing

    Application subnet
        ↓
    10.20.0.0/16 → local
        ↓
    Database subnet

### Future Application Internet Egress

    Private application instance
        ↓
    Application route table
        ↓
    0.0.0.0/0 → AZ-local NAT Gateway
        ↓
    Public subnet
        ↓
    Public route table
        ↓
    0.0.0.0/0 → Internet Gateway
        ↓
    Internet

---

## Design Status

The foundational VPC network has been implemented and validated.

Completed:

- dedicated VPC;
- six-subnet architecture;
- two-Availability-Zone distribution;
- public routing;
- private application routing;
- private database routing;
- Internet Gateway attachment;
- explicit route-table associations.

Deferred by design:

- NAT Gateways;
- application compute;
- load balancing;
- database deployment;
- application security groups;
- workload-specific connectivity.

These components will be introduced when their corresponding requirements are implemented.