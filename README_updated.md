# Snowflake Metadata Hub + AWS Glue Iceberg POC

This repository contains a hands-on Proof of Concept (POC) that demonstrates how Snowflake can discover and query an externally managed Apache Iceberg table stored in Amazon S3 and cataloged in AWS Glue, using:

- AWS Glue Data Catalog
- AWS Glue Iceberg REST endpoint
- AWS Lake Formation
- Catalog-vended credentials
- Snowflake Catalog Integration
- Snowflake Catalog-Linked Database

The Iceberg table remains in AWS. Snowflake does **not** copy the table into Snowflake-managed storage.

---

## What this POC demonstrates

At the end of the setup:

```text
AWS Glue
= Catalog of record for the external Iceberg table

Amazon S3
= Stores Iceberg metadata + Parquet data files

AWS Lake Formation
= Authorizes the Snowflake AWS role and vends temporary S3 credentials

Snowflake
= Discovers the Glue catalog and queries the same external Iceberg table
```

The POC uses:

```text
Catalog-Linked Database
        +
AWS Glue Iceberg REST
        +
ACCESS_DELEGATION_MODE = VENDED_CREDENTIALS
```

A Snowflake External Volume is therefore not used for the table-storage access path in this POC.

---

# Repository Structure

The repository contains three guided setup files:

```text
README.md
Phase-1_AWS_Setup.md
Phase-2_AWS_Setup.md
Phase-3_Snowflake_Setup.md
```

Each Markdown file contains:

- the required commands
- placeholders to update
- an explanation of what each step does
- why the step is required
- validation checks before moving to the next phase

This repository is intentionally designed as a **guided POC**, rather than as a fully automated deployment.

---

# How to Use This Repository

Follow the three files in sequence:

```text
Phase-1_AWS_Setup.md
        |
        v
Phase-2_AWS_Setup.md
        |
        v
Phase-3_Snowflake_Setup.md
```

For each phase:

1. Open the relevant Markdown file.
2. Read the prerequisite section.
3. Replace values shown inside `<...>` placeholders.
4. Copy the command from the code block.
5. Paste and execute it in the appropriate console.
6. Validate the checkpoint before continuing.

Use:

```text
AWS CloudShell / AWS CLI
    for Phase 1 and Phase 2

Snowflake Snowsight
    for the Snowflake SQL in Phase 3

AWS CloudShell / AWS CLI
    for the IAM trust-policy callback inside Phase 3
```

Do not execute all phases together.

The objective is to understand and validate each layer before moving to the next one.

---

# Phase Overview

## Phase 1 — AWS Iceberg Foundation

File:

```text
Phase-1_AWS_Setup.md
```

Phase 1 creates the basic external Iceberg environment:

- Amazon S3 bucket
- AWS Glue database
- Apache Iceberg `customer` table
- sample records using Amazon Athena
- validation of Glue and S3 objects

At the end of Phase 1:

```text
Amazon Athena
      |
      v
AWS Glue Data Catalog
      |
      v
iceberg_poc.customer
      |
      v
Amazon S3
  metadata + Parquet
```

The main goal is to prove that the Iceberg table works correctly **before Snowflake is introduced**.

---

## Phase 2 — AWS Lake Formation and Vended Credentials

File:

```text
Phase-2_AWS_Setup.md
```

Phase 2 prepares the AWS security and credential-vending layer.

It configures:

- `LakeFormationIcebergDataRole`
- Lake Formation registration of the Iceberg S3 location
- verification of the registered S3 location
- full-table external-engine access
- `SnowflakeGlueVendedRole`
- Glue catalog IAM permissions
- `lakeformation:GetDataAccess`
- Lake Formation `DESCRIBE`
- Lake Formation full-table `SELECT`

The most important checkpoint is:

```text
VerificationStatus = VERIFIED
```

Do not continue to Phase 3 if the Lake Formation registration is:

```text
NOT_VERIFIED
```

or:

```text
VERIFICATION_FAILED
```

---

## Phase 3 — Snowflake Setup

File:

```text
Phase-3_Snowflake_Setup.md
```

Phase 3 connects Snowflake to the AWS Glue Iceberg REST catalog.

It includes:

- Snowflake Catalog Integration
- SigV4 authentication
- `ACCESS_DELEGATION_MODE = VENDED_CREDENTIALS`
- retrieval of Snowflake-generated AWS IAM identity information
- AWS IAM trust-policy update
- catalog connectivity validation
- table discovery
- vended-credential validation
- Iceberg metadata lookup
- Catalog-Linked Database creation
- querying the external Iceberg table from Snowflake

There is one AWS callback step inside Phase 3 because Snowflake first generates:

```text
GLUE_AWS_IAM_USER_ARN
GLUE_AWS_EXTERNAL_ID
```

These values must then be added to the AWS trust policy of:

```text
SnowflakeGlueVendedRole
```

After updating the trust policy, return to Snowflake and continue with the validation steps.

---

# Prerequisites

Before starting the POC, make sure the following prerequisites are available.

---

## 1. AWS Account

You need access to an AWS account where you are allowed to create and manage the resources required by this POC.

The commands use:

- Amazon S3
- AWS Glue
- Amazon Athena
- AWS IAM
- AWS Lake Formation
- AWS STS

Use the same AWS Region consistently for the POC.

For example:

```text
<YOUR_AWS_REGION>
```

Replace it with your selected AWS Region.

---

## 2. AWS CLI or AWS CloudShell

The AWS commands can be executed using:

- AWS CloudShell, or
- a local terminal with AWS CLI configured

Before starting, validate the AWS identity:

```bash
aws sts get-caller-identity
```

Confirm that the returned account and IAM identity are the ones you intend to use.

---

## 3. AWS Permissions

The AWS identity executing the commands must have sufficient permissions for the operations used in this POC.

It needs permissions related to:

```text
S3
Glue
Athena
IAM
Lake Formation
STS
```

For a learning POC, an administrative sandbox account is the easiest environment.

For enterprise use, use a controlled deployment role with only the required permissions.

---

## 4. Lake Formation Administrator

The AWS user or role configuring Lake Formation should be a valid Lake Formation Data Lake Administrator, or have equivalent Lake Formation administrative permissions.

This is required to:

- register the S3 location
- configure Lake Formation settings
- grant database permissions
- grant table permissions
- validate registered resources

---

## 5. Amazon Athena

The POC uses Athena to create and populate the Iceberg table.

The walkthrough uses the Athena workgroup:

```text
primary
```

If your environment uses another Athena workgroup, update the commands accordingly.

Athena also needs an S3 output location for query results.

The Phase 1 guide uses:

```text
s3://<YOUR_UNIQUE_S3_BUCKET_NAME>/athena-results/
```

---

## 6. Unique S3 Bucket Name

S3 bucket names must be globally unique.

Before running Phase 1, replace:

```text
<YOUR_UNIQUE_S3_BUCKET_NAME>
```

with a unique bucket name.

The POC stores the Iceberg table under:

```text
s3://<YOUR_UNIQUE_S3_BUCKET_NAME>/iceberg/customer/
```

---

## 7. `jq`

Phase 2 uses `jq` to update the existing Lake Formation data lake settings without replacing unrelated settings.

Validate that `jq` is available:

```bash
jq --version
```

AWS CloudShell normally includes it.

If you execute the commands locally, install `jq` before running the relevant Phase 2 commands.

---

## 8. Snowflake Account

You need access to a Snowflake account where you can create:

- a Catalog Integration
- a Catalog-Linked Database

For simplicity, the Phase 3 walkthrough uses:

```sql
USE ROLE ACCOUNTADMIN;
```

This is acceptable for a learning POC.

For production, use a custom administrative role with only the required privileges instead of using `ACCOUNTADMIN` for normal operations.

---

## 9. Snowflake-to-AWS IAM Trust

Snowflake connects to the AWS Glue Iceberg REST endpoint using SigV4 authentication and the AWS IAM role:

```text
SnowflakeGlueVendedRole
```

After the Snowflake Catalog Integration is created, Snowflake returns:

```text
GLUE_AWS_IAM_USER_ARN
GLUE_AWS_EXTERNAL_ID
```

These values must be used to update the AWS trust policy for `SnowflakeGlueVendedRole`.

This callback step is mandatory.

---

# Important Placeholders

The setup files intentionally use placeholders instead of real account-specific values.

Replace the relevant values before executing the commands.

Common placeholders include:

```text
<YOUR_AWS_REGION>
<YOUR_UNIQUE_S3_BUCKET_NAME>
<AWS_ACCOUNT_ID>
<GLUE_DATABASE>
<GLUE_AWS_IAM_USER_ARN_FROM_SNOWFLAKE>
<GLUE_AWS_EXTERNAL_ID_FROM_SNOWFLAKE>
```

Some AWS commands retrieve the AWS account ID automatically using:

```bash
aws sts get-caller-identity --query Account --output text
```

So always follow the variable setup shown in the relevant phase rather than manually replacing every account ID.

---

# POC Object Names

The walkthrough uses the following names consistently:

```text
Glue Database
iceberg_poc

Iceberg Table
customer

Lake Formation Registration Role
LakeFormationIcebergDataRole

Snowflake-to-AWS IAM Role
SnowflakeGlueVendedRole

Snowflake Catalog Integration
GLUE_ICEBERG_REST_VENDED_INT

Snowflake Catalog-Linked Database
ICEBERG_GLUE_VENDED_DB
```

You can change these names, but if you do, update them consistently across all three phases.

---

# Security Model

There are two AWS IAM roles because they perform different jobs.

## `SnowflakeGlueVendedRole`

This is the IAM role Snowflake assumes.

It allows Snowflake to:

```text
Access the required AWS Glue catalog metadata
        +
Call AWS Lake Formation
        +
Request temporary data access
```

In simple language:

> This role represents Snowflake when Snowflake talks to AWS.

---

## `LakeFormationIcebergDataRole`

This is the role Lake Formation uses for the registered S3 location.

It allows Lake Formation to reach the Iceberg objects stored in S3.

In simple language:

> This role allows Lake Formation to access the registered S3 location.

---

The overall flow is:

```text
                        Snowflake
                           |
                           v
                 AWS Glue Iceberg REST
                    /               \
                   /                 \
                  v                   v
       AWS Glue Data Catalog     Lake Formation
       "What/where is table?"    "Can this role access it?"
                  |                   |
                  |             full-table check
                  |                   |
                  |             temporary scoped
                  |             S3 credentials
                  |                   |
                  \______       ______/
                         \     /
                          v   v
                        Snowflake
                           |
                           | reads directly
                           v
                        Amazon S3
                  Iceberg metadata + Parquet
```

These are not two duplicate data paths.

The Glue side is the **metadata path**.

The Lake Formation side is the **authorization and credential-vending path**.

Snowflake then uses the metadata location and the temporary credentials to read S3 directly.

---

# Lake Formation Permission Boundary

Lake Formation authorizes the AWS IAM role used by Snowflake.

It does not replace Snowflake RBAC for individual Snowflake users.

The model is:

```text
Snowflake User
      |
      v
Snowflake RBAC / policies
      |
      v
SnowflakeGlueVendedRole
      |
      v
Lake Formation full-table authorization
```

For this POC, Lake Formation access is granted at full-table level to the AWS IAM role.

User-level authorization inside Snowflake should still be controlled using Snowflake RBAC and supported Snowflake governance policies.

---

# Execution Checkpoints

## Phase 1 Checkpoint

Before starting Phase 2, confirm:

```text
S3 bucket exists                         ✅
Bucket is private                        ✅
S3 encryption is enabled                 ✅
Glue database exists                     ✅
Iceberg table exists                     ✅
Athena SELECT returns sample rows        ✅
Iceberg files are visible in S3          ✅
```

Expected logical sample rows:

```text
1 | Customer A | Agra   | 15000.00
2 | Customer B | Delhi  | 25000.00
3 | Customer C | Mumbai | 18000.00
```

---

## Phase 2 Checkpoint

Before starting the Snowflake configuration, confirm:

```text
LakeFormationIcebergDataRole exists               ✅
S3 access policy is attached                      ✅
Iceberg S3 location is registered                 ✅
VerificationStatus = VERIFIED                     ✅
Glue table is registered with Lake Formation      ✅
Full-table external-engine access is enabled      ✅
SnowflakeGlueVendedRole exists                    ✅
Glue/Lake Formation IAM permissions are attached  ✅
lakeformation:GetDataAccess is granted            ✅
Lake Formation DESCRIBE is granted                ✅
Lake Formation full-table SELECT is granted       ✅
```

The most important validation is:

```text
VerificationStatus = VERIFIED
```

Do not continue if this check fails.

---

## Phase 3 Checkpoint

At the end of Phase 3, confirm:

```text
Snowflake Catalog Integration exists                  ✅
Final AWS trust relationship is configured            ✅
Snowflake can verify the catalog                       ✅
Snowflake can list the Glue table                      ✅
Snowflake can retrieve vended credentials              ✅
Snowflake can resolve the current Iceberg metadata     ✅
Catalog-Linked Database exists                         ✅
Glue namespace is visible in Snowflake                 ✅
Iceberg customer table is visible                      ✅
Snowflake SELECT returns the sample rows               ✅
```

---

# Troubleshooting Order

If something fails, troubleshoot in this order:

```text
Can Athena query the Iceberg table?
        |
        v
Can Glue see the table?
        |
        v
Is the S3 location VERIFIED in Lake Formation?
        |
        v
Does SnowflakeGlueVendedRole have the required IAM permissions?
        |
        v
Does it have Lake Formation DESCRIBE + full-table SELECT?
        |
        v
Is AllowFullTableExternalDataAccess enabled?
        |
        v
Is the final Snowflake IAM trust policy correct?
        |
        v
Can Snowflake authenticate to Glue Iceberg REST?
        |
        v
Can Snowflake retrieve vended credentials?
        |
        v
Can Snowflake load the Iceberg metadata?
        |
        v
Can Snowflake query the table?
```

This order helps isolate the failing layer instead of troubleshooting everything at once.

---

# Read-Only Recommendation

This POC is mainly intended to demonstrate:

```text
Discovery + Query
```

A Catalog-Linked Database can support write operations depending on the configuration and supported catalog capabilities.

Before using this pattern in production, explicitly review the allowed write behavior and configure the linked database according to your read/write requirements.

For a read-only architecture, do not rely on defaults without reviewing the Snowflake write-operation settings.

---

# Cost Considerations

This POC can create billable usage in both AWS and Snowflake.

Possible AWS charges include:

- S3 storage
- Athena queries
- Glue usage
- related AWS service usage

Possible Snowflake charges can include:

- cloud-services usage for catalog discovery and synchronization
- compute or metadata-refresh-related operations depending on the activity

Use a sandbox environment where possible and remove resources after completing the POC if they are no longer required.

---

# Recommended POC Environment

For the easiest learning experience:

```text
AWS sandbox account
AWS CloudShell
One AWS Region
Small sample Iceberg table
Snowflake non-production account
ACCOUNTADMIN only for the POC setup
```

Avoid starting with production S3 buckets or production Glue databases.

First validate the full end-to-end flow with a small disposable dataset.

---

# What This POC Proves

At the end of the exercise:

```text
Amazon S3
still stores the table

AWS Glue
still owns the external catalog

AWS Lake Formation
controls AWS-side full-table authorization and credential vending

Snowflake
discovers the catalog and queries the same Iceberg files
```

The key architectural outcome is:

> Snowflake can discover and query an externally managed Apache Iceberg table through AWS Glue and Lake Formation without first copying the table into Snowflake-managed storage.

This demonstrates the inbound external-catalog-to-Snowflake side of a broader metadata-hub architecture.

---

# Companion Article

This repository is designed to accompany the article:

**Snowflake as a Metadata Hub for Open Data — Concept and a Hands-On AWS Glue + Iceberg POC**

After publishing the article, add the link here:

```text
<YOUR_MEDIUM_ARTICLE_URL>
```

The article explains the architecture and concepts, while this repository provides the step-by-step implementation.

---

# Disclaimer

This repository is intended for learning and POC purposes.

Before using this architecture in production, review:

- IAM least-privilege design
- Lake Formation governance
- Snowflake RBAC
- encryption requirements
- networking requirements
- write permissions
- catalog synchronization behavior
- cost controls
- operational monitoring
- organizational security standards

Always validate current AWS and Snowflake documentation before using the pattern in a production environment.
