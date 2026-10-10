# Project Nimbus — Security Architecture

## Overview

This document defines the security and access architecture for Project Nimbus, the AWS infrastructure supporting NimbusDesk.

The security model follows a layered approach combining:

- network segmentation;
- security groups;
- AWS Identity and Access Management (IAM);
- workload identities;
- secrets management;
- encryption;
- controlled administrative access;
- least privilege.

Security controls are introduced according to workload requirements rather than granting broad permissions in advance.

---

## Security Objectives

The Nimbus security architecture must:

- expose only the required public application entry point;
- keep application compute private;
- keep database resources private;
- restrict communication between application tiers;
- avoid direct SSH exposure for application instances;
- use IAM roles instead of long-lived AWS credentials for workloads;
- protect privileged administrative identities with MFA;
- prevent production secrets from entering source control;
- encrypt sensitive data in transit and at rest;
- maintain clear separation between human and workload identities;
- support future auditing and operational investigation.

---

## Trust Model

NimbusDesk uses progressively restricted trust boundaries.

    Internet
        │
        │ HTTPS / TCP 443
        ▼
    Public Entry Layer
    Application Load Balancer
        │
        │ Application traffic
        ▼
    Private Application Layer
    EC2 / Auto Scaling
        │
        │ PostgreSQL / TCP 5432
        ▼
    Private Data Layer
    RDS PostgreSQL

The public entry layer is the intended Internet-facing application boundary.

Application compute and database resources are designed to remain private.

---

## Security Group Architecture

Three primary security groups establish the workload trust relationships:

    nimbus-alb-sg
    nimbus-app-sg
    nimbus-db-sg

### ALB Security Group

    Name: nimbus-alb-sg

Current inbound rule:

    HTTPS
    TCP 443
    Source: 0.0.0.0/0

Purpose:

Permit public HTTPS traffic to the future NimbusDesk Application Load Balancer.

The security group permits traffic on TCP 443 but does not itself provide TLS. TLS termination will be configured on the future ALB using an HTTPS listener and an ACM certificate.

---

### Application Security Group

    Name: nimbus-app-sg

Current inbound rules:

    None

The application listening port has intentionally not been defined because NimbusDesk application deployment has not yet established the production listening port.

When the application port is known, the intended relationship is:

    Source: nimbus-alb-sg
    Protocol: TCP
    Port: <NimbusDesk application port>

The application port will not be exposed directly to 0.0.0.0/0.

No inbound SSH rule is currently permitted.

---

### Database Security Group

    Name: nimbus-db-sg

Inbound rule:

    PostgreSQL
    TCP 5432
    Source: nimbus-app-sg

This creates a workload-level trust relationship between the application and database tiers.

The database rule does not permit PostgreSQL access from:

    0.0.0.0/0

or broadly from:

    10.20.0.0/16

Instead, access is restricted to resources associated with the Nimbus application security group.

---

## Routing and Security

Routing and security groups solve different problems.

Routing establishes whether a network path exists.

Security groups determine whether traffic using that path is permitted.

For application-to-database communication:

    Application subnet
        │
        │ 10.20.0.0/16 → local
        ▼
    Database subnet

provides the VPC network path.

The security relationship:

    nimbus-app-sg
        │
        │ TCP 5432
        ▼
    nimbus-db-sg

determines whether the PostgreSQL connection is permitted at the security-group layer.

A valid route therefore does not by itself guarantee successful application connectivity.

---

## Security Group State

AWS security groups are stateful.

When an allowed connection is established, response traffic for that connection is automatically permitted by the security group state tracking.

This differs from Network ACLs, which operate at subnet level and are stateless.

No custom Network ACL architecture is currently required for Nimbus.

---

## Administrative EC2 Access

Traditional inbound SSH access is not part of the current Nimbus architecture.

The target administration mechanism is:

    Administrator
        │
        ▼
    AWS Systems Manager
    Session Manager
        │
        ▼
    Private EC2

This avoids requiring:

- inbound TCP 22;
- public EC2 addresses for administration;
- externally exposed SSH services;
- application-server SSH key distribution.

Systems Manager connectivity will be validated when EC2 instances are deployed.

Because the application subnets currently have no NAT Gateway, the required network path to Systems Manager must be deliberately provided when compute is introduced.

Possible connectivity mechanisms will be evaluated against the deployed architecture rather than assumed in advance.

---

## Workload Identity

NimbusDesk EC2 instances will use an IAM role rather than long-lived AWS access keys.

The workload role is:

    nimbus-ec2-role

Trusted service:

    ec2.amazonaws.com

The role trust relationship permits the EC2 service to assume the role using:

    sts:AssumeRole

Initial permission:

    AmazonSSMManagedInstanceCore

This provides the core instance-side permissions required for AWS Systems Manager integration.

---

## Least-Privilege Expansion

The workload role will be expanded only when new application requirements exist.

Expected future permissions include restricted access to:

- the NimbusDesk S3 attachment bucket;
- the specific NimbusDesk database secret;
- other AWS services only where application requirements justify them.

Broad workload policies such as:

    AdministratorAccess
    AmazonS3FullAccess

are not part of the Nimbus workload security design.

---

## Human and Workload Identity Separation

Nimbus separates three identity contexts.

### Root Identity

The AWS root identity is reserved for account-level, recovery, or root-specific operations.

MFA is enabled.

### Administrative Identity

    nimbus-admin

is used for day-to-day infrastructure administration.

MFA is enabled.

The administrative user currently has AdministratorAccess because Project Nimbus is being developed within a dedicated learning environment.

This administrative permission model must not be inherited by application workloads.

### Workload Identity

    nimbus-ec2-role

represents the application compute identity.

Its permissions are independent of the human administrator's permissions and are expanded according to workload requirements.

---

## Secrets Management

Production secrets must not be committed to source control or hardcoded into the NimbusDesk application.

The target architecture uses AWS Secrets Manager for sensitive production database credentials.

Future flow:

    AWS Secrets Manager
            │
            │ authorised retrieval
            ▼
    nimbus-ec2-role
            │
            ▼
    NimbusDesk
            │
            ▼
    RDS PostgreSQL

Access will be restricted to the specific required secret rather than granting broad Secrets Manager access.

Secrets Manager resources have not yet been created because the production database and credentials do not yet exist.

---

## Configuration Classification

| Configuration | Planned Handling |
|---|---|
| Database password | AWS Secrets Manager |
| Database username | Stored with database secret |
| Database endpoint | Runtime configuration |
| Database port | Runtime configuration |
| Database name | Runtime configuration |
| AWS Region | Runtime/deployment configuration |
| S3 bucket name | Runtime/deployment configuration |
| AWS access key | Not stored — IAM role |
| AWS secret access key | Not stored — IAM role |

Local development may use an `.env` file.

The application repository may provide an `.env.example` containing placeholders, but production credentials and local `.env` files must not be committed.

---

## Encryption Baseline

Nimbus requires encryption both in transit and at rest where applicable.

### Internet Traffic

Target:

    Client → HTTPS/TLS → ALB

TLS will be implemented through an ALB HTTPS listener and AWS Certificate Manager when the public application endpoint and domain are deployed.

### EBS

EBS encryption by default has been enabled for the Nimbus AWS Region:

    Region: eu-west-2
    Default EBS encryption: Enabled

This provides a regional guardrail so newly created EBS volumes are encrypted by default.

### RDS

Target requirements:

    Public accessibility: Disabled
    Storage encryption: Enabled
    Application connection: TLS

These controls will be enforced when RDS PostgreSQL is provisioned.

### S3

The future attachment bucket will:

- use server-side encryption;
- retain S3 Block Public Access;
- use IAM-controlled application access;
- use HTTPS for application communication.

### Secrets Manager

Production secrets stored in Secrets Manager will use the service's encryption capabilities and be accessed through authorised AWS API calls.

---

## Auditing

Operational and administrative auditing will be expanded during the observability and auditing stage of Project Nimbus.

AWS CloudTrail will form part of the target audit architecture for AWS account and API activity.

This control is not currently claimed as implemented within this phase.

---

## Security Design Principle

Project Nimbus follows:

    Required communication
            ↓
    Minimum necessary network access
            ↓
    Minimum necessary AWS permissions
            ↓
    No unnecessary public exposure
            ↓
    Encrypt sensitive data
            ↓
    Observe and audit
            ↓
    Reassess as workload requirements evolve

Security controls will continue to evolve as the application, data services and operational requirements become measurable.