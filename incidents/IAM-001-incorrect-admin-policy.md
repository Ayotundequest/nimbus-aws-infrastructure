# IAM-001 — Administrative User Unable to Create VPC

## Summary

During the initial Project Nimbus AWS setup, the nimbus-admin IAM user could access portions of the AWS console but was unable to create the Nimbus VPC.

---

## Symptoms

Initial AWS API operations returned authorization failures.

One observed failure was:

    ec2:CreateVpc

AWS reported that no identity-based policy allowed the requested action.

---

## Scope

The issue affected the administrative IAM identity used for Project Nimbus.

The AWS account itself remained accessible through the root identity.

---

## Investigation

The error showed that authentication had succeeded because AWS identified the requesting principal as the nimbus-admin IAM user.

The failure occurred during authorization.

The IAM permissions attached to nimbus-admin were therefore inspected.

---

## Root Cause

The AWS-managed policy:

    AdministratorAccess-Amplify

had been selected instead of:

    AdministratorAccess

The Amplify-specific policy did not provide the EC2/VPC administrative permissions required to create the Nimbus VPC.

---

## Resolution

The incorrect policy was removed and the intended AWS-managed AdministratorAccess policy was attached to the administrative IAM user.

Normal engineering activity was then resumed using nimbus-admin rather than the AWS root user.

---

## Validation

After the permission correction, nimbus-admin successfully created:

    nimbus-prod-vpc
    10.20.0.0/16

The VPC creation operation completed successfully.

---

## Engineering Lesson

Successful authentication does not imply successful authorization.

The incident demonstrated the distinction between:

    Authentication
    "Who are you?"

and:

    Authorization
    "What are you permitted to do?"

AWS policy names must also be verified carefully rather than selected based on similar naming.

The root identity remains reserved for account-level or recovery operations rather than routine infrastructure administration.