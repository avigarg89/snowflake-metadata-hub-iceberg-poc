Snowflake Metadata Hub + AWS Glue Iceberg POC
This repository contains a hands-on Proof of Concept (POC) that demonstrates how Snowflake can discover and query an externally managed Apache Iceberg table stored in Amazon S3 and cataloged in AWS Glue, using:
AWS Glue Data Catalog
AWS Glue Iceberg REST endpoint
AWS Lake Formation
Catalog-vended credentials
Snowflake Catalog Integration
Snowflake Catalog-Linked Database
The POC keeps the Iceberg data in AWS. Snowflake does not copy the table into Snowflake-managed storage.

What this POC demonstrates
At the end of the setup:
AWS Glue
= catalog of record for the Iceberg table

Amazon S3
= stores Iceberg metadata + Parquet files

AWS Lake Formation
= authorizes the Snowflake AWS role and vends temporary S3 credentials

Snowflake
= discovers the Glue catalog through Iceberg REST and queries the same Iceberg table
The POC uses:
Catalog-Linked Database
        +
AWS Glue Iceberg REST
        +
ACCESS_DELEGATION_MODE = VENDED_CREDENTIALS
A Snowflake External Volume is therefore not used for the table-storage access path in this POC.

Repository Structure
Execute the files in this order:
Phase-1_AWS_Setup.md
Phase-2_AWS_Setup.md
Phase-3_Snowflake_Setup.md
Phase 1 — AWS Setup
Creates the basic Iceberg environment:
Amazon S3 bucket
AWS Glue database
Apache Iceberg customer table
Sample data using Amazon Athena
Validation of Glue and S3 objects
Phase 2 — AWS Setup
Creates the security and credential-vending setup:
Lake Formation S3 registration
LakeFormationIcebergDataRole
SnowflakeGlueVendedRole
IAM permissions
Lake Formation database/table permissions
Full-table external-engine access
Verification of the registered S3 location
Phase 3 — Snowflake Setup
Connects Snowflake to AWS:
Snowflake Glue Iceberg REST Catalog Integration
Snowflake-generated AWS IAM user ARN and External ID
Final AWS trust relationship
Catalog connectivity tests
Vended-credential test
Catalog-Linked Database
Querying the external Iceberg table from Snowflake

Prerequisites
Before starting the POC, make sure the following prerequisites are available.

1. AWS Account
You need access to an AWS account where you are allowed to create and manage the resources used in this POC.
The scripts use the following AWS services:
Amazon S3
AWS Glue
Amazon Athena
AWS IAM
AWS Lake Formation
AWS STS
You should know which AWS Region you want to use.
Use the same AWS Region consistently for:
S3
Glue
Athena
Lake Formation
Example:
<AWS_REGION>
Replace this placeholder with your selected AWS Region, for example:
us-west-2
Before starting, confirm that the AWS services and Glue Iceberg REST capability required for the POC are available in the Region you choose.

2. AWS CLI Access
The AWS CLI must already be authenticated.
You can use:
AWS CloudShell, or
your local terminal with AWS CLI configured
Validate your identity before starting:
aws sts get-caller-identity
You should see the AWS account and IAM identity you intend to use.

3. AWS Permissions
The identity running the setup must have enough permissions to create and manage the POC resources.
At a minimum, it needs permissions for the operations used in the scripts across:
S3
Glue
Athena
IAM
Lake Formation
STS
For a POC, using an administrative sandbox account is the simplest option.
For an enterprise environment, use a controlled deployment role with only the required privileges.

4. Lake Formation Administrator
The user or role configuring Lake Formation should be a valid Lake Formation Data Lake Administrator or have equivalent Lake Formation administrative permissions.
This is needed to:
register the S3 location
configure Lake Formation settings
grant database permissions
grant table permissions
validate Lake Formation resources
Do not rely on the AWS root user as your normal Lake Formation administrator.

5. Amazon Athena
The POC uses Athena to create and populate the Iceberg table.
The default Athena workgroup used in the scripts is:
primary
If your organization uses another workgroup, update the commands accordingly.
Athena also needs an S3 output location for query results.
The Phase 1 script creates/uses:
s3://<YOUR_S3_BUCKET>/athena-results/

6. Unique S3 Bucket Name
S3 bucket names must be globally unique.
Before running Phase 1, choose a unique bucket name and replace:
<YOUR_UNIQUE_S3_BUCKET_NAME>
Example:
mycompany-snowflake-iceberg-poc
The POC stores Iceberg files under:
s3://<YOUR_UNIQUE_S3_BUCKET_NAME>/iceberg/customer/

7. jq Command-Line Utility
Phase 2 uses jq to safely update the Lake Formation data lake settings JSON.
If you use AWS CloudShell, jq is normally available.
If you run the scripts locally, verify:
jq --version
If it is not installed, install it before running Phase 2.

8. Snowflake Account
You need access to a Snowflake account where you can create:
Catalog Integration
Catalog-Linked Database
For this POC, the Snowflake setup uses:
USE ROLE ACCOUNTADMIN;
This is done only to keep the POC simple.
For a production environment, use a custom administrative role with the required privileges instead of using ACCOUNTADMIN for normal operations.

9. Snowflake Access to AWS Glue Iceberg REST
The POC connects Snowflake to the AWS Glue Iceberg REST endpoint using:
SIGV4
and an AWS IAM role created in Phase 2:
SnowflakeGlueVendedRole
Phase 3 creates the Snowflake catalog integration and then retrieves:
GLUE_AWS_IAM_USER_ARN
GLUE_AWS_EXTERNAL_ID
These values must be copied back into the AWS trust policy for SnowflakeGlueVendedRole.
This AWS callback step is mandatory.

Important Placeholders
The scripts intentionally use placeholders instead of real account details.
Replace these values before executing the commands:
<YOUR_AWS_REGION>
<YOUR_UNIQUE_S3_BUCKET_NAME>
<AWS_ACCOUNT_ID>
<GLUE_DATABASE>
<GLUE_AWS_IAM_USER_ARN_FROM_SNOWFLAKE>
<GLUE_AWS_EXTERNAL_ID_FROM_SNOWFLAKE>
Most scripts automatically retrieve the AWS Account ID using:
aws sts get-caller-identity --query Account --output text
So in many AWS commands you do not need to type the account ID manually.

Important POC Names
The repository uses the following names consistently:
Glue Database
iceberg_poc

Iceberg Table
customer

Lake Formation Registration Role
LakeFormationIcebergDataRole

Snowflake-to-AWS Role
SnowflakeGlueVendedRole

Snowflake Catalog Integration
GLUE_ICEBERG_REST_VENDED_INT

Snowflake Catalog-Linked Database
ICEBERG_GLUE_VENDED_DB
You can rename them, but if you do, update the names consistently across all three phases.

Execution Order
Do not run all three phases at once.
Run and validate one phase before moving to the next.

Phase 1 Checkpoint
Before starting Phase 2, confirm:
S3 bucket exists                         ✅
Glue database exists                     ✅
Iceberg table exists                     ✅
Athena can query the table               ✅
Iceberg metadata files exist in S3       ✅
Parquet files exist in S3                ✅
The sample table should return:
1 | Customer A | Agra   | 15000.00
2 | Customer B | Delhi  | 25000.00
3 | Customer C | Mumbai | 18000.00

Phase 2 Checkpoint
Before starting Snowflake configuration, confirm:
LakeFormationIcebergDataRole exists              ✅
S3 location is registered with Lake Formation    ✅
VerificationStatus = VERIFIED                    ✅
Glue table is registered with Lake Formation     ✅
Full-table external-engine access is enabled     ✅
SnowflakeGlueVendedRole exists                   ✅
lakeformation:GetDataAccess is granted           ✅
Lake Formation DESCRIBE permission is granted    ✅
Lake Formation full-table SELECT is granted      ✅
The most important validation is:
VerificationStatus = VERIFIED
Do not continue if the Lake Formation registration is:
NOT_VERIFIED
or:
VERIFICATION_FAILED

Phase 3 Checkpoint
At the end of Phase 3, confirm:
Catalog Integration exists                       ✅
AWS trust relationship is updated                ✅
Snowflake can verify the catalog                  ✅
Snowflake can list Glue tables                    ✅
Snowflake can retrieve vended credentials         ✅
Snowflake can load the Iceberg metadata           ✅
Catalog-Linked Database exists                   ✅
Snowflake can query the external Iceberg table    ✅

Security Model Used in This POC
There are two AWS IAM roles because they solve different problems.
SnowflakeGlueVendedRole
Used by Snowflake to communicate with:
AWS Glue
AWS Lake Formation
In simple language:
This role represents Snowflake when Snowflake talks to AWS.

LakeFormationIcebergDataRole
Used by Lake Formation to access:
Amazon S3
In simple language:
This role allows Lake Formation to reach the registered Iceberg storage location.

The request flow is:
Snowflake
   |
   v
SnowflakeGlueVendedRole
   |
   +------> AWS Glue Data Catalog
   |        returns table metadata
   |
   +------> AWS Lake Formation
            checks full-table permission
                    |
                    v
            temporary S3 credentials
                    |
                    v
                 Snowflake
                    |
                    v
                 Amazon S3
         Iceberg metadata + Parquet
Snowflake reads S3 directly using the temporary credentials.

Lake Formation Permission Boundary
Lake Formation authorizes the AWS IAM role used by Snowflake.
It does not replace Snowflake RBAC for individual Snowflake users.
The model is:
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
For this POC, Lake Formation access should be full-table and unfiltered.
Snowflake user-level controls should be implemented in Snowflake.

Read-Only Recommendation
This POC is primarily intended to demonstrate:
Discovery + Query
Catalog-linked databases can support write operations to the remote catalog.
For a read-only implementation, review the Snowflake catalog-linked database setting:
ALLOWED_WRITE_OPERATIONS = NONE
before using the pattern in a production environment.

Cost Considerations
This POC can create billable resources or usage in both AWS and Snowflake.
Possible AWS charges include:
S3 storage
Athena queries
Glue usage
Lake Formation-related underlying service usage
Possible Snowflake charges include:
cloud-services usage for catalog discovery/synchronization
compute or refresh-related usage depending on the operation
Use a sandbox account where possible and clean up the resources after completing the POC.

Recommended Environment
For the easiest learning experience:
AWS sandbox account
AWS CloudShell
One AWS Region
Snowflake non-production account
ACCOUNTADMIN for POC only
Small sample Iceberg table
Do not start with an enterprise production bucket or production Glue database.
First validate the complete flow with a small disposable dataset.

Troubleshooting Order
If something fails, troubleshoot in this order:
Can Athena query the Iceberg table?
        |
        v
Can Glue see the table?
        |
        v
Is the S3 location VERIFIED in Lake Formation?
        |
        v
Does SnowflakeGlueVendedRole have IAM access?
        |
        v
Does it have Lake Formation DESCRIBE + SELECT?
        |
        v
Can Snowflake authenticate to Glue REST?
        |
        v
Can Snowflake retrieve vended credentials?
        |
        v
Can Snowflake load the Iceberg metadata?
        |
        v
Can Snowflake query the table?
This order helps isolate the exact layer causing the issue.

Companion Article
The concepts behind this POC are explained separately in the Medium article:
Snowflake as a Metadata Hub for Open Data — Concept and a Hands-On AWS Glue + Iceberg POC
Add your Medium article URL here after publishing:
<YOUR_MEDIUM_ARTICLE_URL>

Disclaimer
This repository is intended for learning and POC purposes.
Before using the pattern in production, review:
IAM least-privilege requirements
Lake Formation governance model
Snowflake RBAC
write permissions
encryption requirements
networking requirements
cost controls
catalog synchronization behavior
your organization's security standards
