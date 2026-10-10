# Project Nimbus — Security & Access Implementation

## Objective

Establish the initial security and access foundation for the NimbusDesk AWS environment before application compute and data services are deployed.

The implementation focused on:

- workload trust boundaries;
- security groups;
- IAM workload identity;
- administrative identity separation;
- secure EC2 administration strategy;
- secrets-management strategy;
- encryption baseline.

---

## Environment

    Cloud Provider: AWS
    Region: eu-west-2
    Environment: Production-oriented lab
    VPC: nimbus-prod-vpc

---

## Security Groups

Three security groups were created inside the Nimbus production VPC.

### nimbus-alb-sg

Purpose:

Protect the future public Application Load Balancer.

Inbound:

    HTTPS
    TCP 443
    Source: 0.0.0.0/0

The rule permits public HTTPS traffic to the future ALB.

No ALB currently exists, therefore this security group is not yet attached to a workload.

---

### nimbus-app-sg

Purpose:

Protect the future private NimbusDesk EC2 application instances.

Inbound:

    None

No application-port rule was created because the production application listening port has not yet been established.

When the application is deployed, inbound application traffic will be permitted from `nimbus-alb-sg` rather than from the public Internet.

No inbound SSH rule was added.

---

### nimbus-db-sg

Purpose:

Protect the future RDS PostgreSQL database.

Inbound:

    PostgreSQL
    TCP 5432
    Source: nimbus-app-sg

This restricts database network access to resources associated with the application security group.

The database is not exposed using a broad VPC CIDR or Internet source.

---

## EC2 Administration Strategy

AWS Systems Manager Session Manager was selected as the target administration method for Nimbus application instances.

This removes the architectural requirement for inbound SSH access to the private application tier.

The design therefore currently contains:

    TCP 22 inbound: None

The operational SSM connection will be validated when EC2 compute is deployed.

---

## EC2 IAM Role

An IAM role was created:

    nimbus-ec2-role

Trusted entity:

    ec2.amazonaws.com

The trust relationship permits:

    sts:AssumeRole

Initial attached policy:

    AmazonSSMManagedInstanceCore

The role will eventually be attached to NimbusDesk EC2 instances.

No application AdministratorAccess permission was granted.

No broad S3 or Secrets Manager permissions were granted.

---

## Workload Credentials

No IAM user or long-lived AWS access key was created for the NimbusDesk application.

AWS service access from EC2 will use temporary credentials provided through the instance's IAM role.

Additional permissions will be introduced only when the corresponding resources and application requirements exist.

---

## Secrets Strategy

AWS Secrets Manager was selected for future production database credentials.

No production database secret has been created yet because the RDS database does not currently exist.

When implemented, `nimbus-ec2-role` will receive permission to retrieve only the required Nimbus database secret.

Production secrets must not be committed to Git.

---

## Administrative Access Review

Administrative identity controls were reviewed.

### Root

    MFA: Enabled
    Intended use: Root-specific/account-level operations only

### nimbus-admin

    MFA: Enabled
    Intended use: Day-to-day Nimbus infrastructure administration

The application workload does not inherit the administrator's permissions.

---

## EBS Encryption

EBS encryption by default was reviewed in the Nimbus deployment Region.

Initial state:

    Disabled

The regional account setting was changed to:

    Enabled

Result:

New EBS volumes created in `eu-west-2` are encrypted by default.

The AWS-managed EBS encryption key is currently sufficient for the project requirements. No customer-managed KMS key requirement has yet been identified.

---

## Deferred Security Controls

The following controls require resources that have not yet been deployed.

### Application Load Balancer

Deferred:

- HTTPS listener;
- ACM certificate;
- final ALB-to-application security-group rule.

Reason:

The ALB, domain and application listening port do not yet exist.

### RDS

Deferred:

- public-access configuration;
- storage encryption validation;
- TLS database connectivity;
- database credentials;
- Secrets Manager integration.

Reason:

RDS PostgreSQL has not yet been provisioned.

### S3

Deferred:

- attachment bucket;
- Block Public Access validation;
- server-side encryption validation;
- least-privilege object permissions.

Reason:

The attachment storage layer has not yet been implemented.

### Systems Manager Connectivity

Deferred:

- managed-instance validation;
- Session Manager connection testing;
- private subnet connectivity to SSM services.

Reason:

Application EC2 instances do not yet exist.

### AWS Auditing

Deferred:

- Project Nimbus CloudTrail/audit implementation and validation.

Reason:

Auditing will be implemented during the appropriate operational/observability stage.

---

## Validation

| Control | Result |
|---|---|
| ALB SG permits HTTPS only for current public requirement | PASS |
| Application SG has no public inbound rule | PASS |
| Application SG has no inbound SSH rule | PASS |
| Database SG permits PostgreSQL from application SG | PASS |
| Database SG does not permit PostgreSQL from Internet | PASS |
| EC2 workload role exists | PASS |
| EC2 is trusted to assume workload role | PASS |
| SSM core policy attached | PASS |
| Workload AdministratorAccess absent | PASS |
| Broad S3 workload access absent | PASS |
| Root MFA enabled | PASS |
| Administrative IAM MFA enabled | PASS |
| EBS encryption by default enabled in eu-west-2 | PASS |
| Production secrets committed to repository | NO |
| RDS-specific controls | DEFERRED |
| S3-specific controls | DEFERRED |
| ALB TLS implementation | DEFERRED |
| SSM operational connectivity | DEFERRED |

---

## Implementation Outcome

The initial Nimbus security foundation has been established without granting permissions or exposing services before they are required.

The implemented environment now provides:

- explicit security-group trust boundaries;
- restricted database access architecture;
- no direct SSH exposure for future application instances;
- separate human and workload identities;
- an EC2 IAM role for Systems Manager integration;
- MFA-protected privileged identities;
- regional default EBS encryption;
- defined secrets-management and encryption strategies.

Security controls requiring ALB, EC2, RDS, S3 or application resources will be implemented and validated alongside those resources.

The security architecture therefore remains intentionally incremental and workload-driven.