# Project Nimbus — Network Implementation

## Objective

Implement the foundational AWS network required to support the NimbusDesk production architecture.

---

## Environment

    Cloud Provider: AWS
    Region: eu-west-2
    Environment: Production
    VPC: nimbus-prod-vpc
    VPC CIDR: 10.20.0.0/16

---

## Implementation Summary

A dedicated VPC was created manually rather than using the automated "VPC and more" workflow.

This allowed each network component and relationship to be implemented and validated individually.

Six subnets were created across two Availability Zones:

    eu-west-2a
    ├── nimbus-public-a  10.20.1.0/24
    ├── nimbus-app-a     10.20.11.0/24
    └── nimbus-db-a      10.20.21.0/24

    eu-west-2b
    ├── nimbus-public-b  10.20.2.0/24
    ├── nimbus-app-b     10.20.12.0/24
    └── nimbus-db-b      10.20.22.0/24

An Internet Gateway was created and attached to the VPC.

A dedicated public route table provides:

    10.20.0.0/16 → local
    0.0.0.0/0    → Internet Gateway

Only the two public subnets are associated with this route table.

Dedicated application and database route tables were created and explicitly associated with their respective subnets.

The private route tables currently contain only:

    10.20.0.0/16 → local

NAT Gateway provisioning was intentionally deferred until an application workload exists that requires outbound internet access.

---

## Validation

The implementation was validated against the approved network design.

| Validation | Result |
|---|---|
| VPC uses 10.20.0.0/16 | PASS |
| Six required subnets exist | PASS |
| Subnets span two Availability Zones | PASS |
| CIDR allocations do not overlap | PASS |
| Internet Gateway attached to Nimbus VPC | PASS |
| Public subnets explicitly associated with public route table | PASS |
| Public default route targets Internet Gateway | PASS |
| Application subnets use dedicated private route tables | PASS |
| Database subnets use dedicated private route tables | PASS |
| Application subnets have no direct IGW route | PASS |
| Database subnets have no default internet route | PASS |
| NAT provisioning deferred and documented | PASS |

---

## Routing Behaviour

Resources inside the VPC can use the local VPC route to reach addresses within 10.20.0.0/16, subject to applicable security controls.

For example, an application resource in 10.20.12.0/24 can be routed toward a database resource in 10.20.21.0/24 using:

    10.20.0.0/16 → local

A separate route for 10.20.21.0/24 is not required.

The application subnets currently cannot route general internet-bound traffic because no default route exists in their route tables.

This is expected behaviour until the outbound NAT architecture is implemented.

---

## Engineering Outcome

The implemented network establishes the required public, application and database boundaries without prematurely provisioning workload or capacity-dependent infrastructure.

The environment is ready for the next project stage: security and access controls.