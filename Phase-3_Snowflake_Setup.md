# Phase 3 — Snowflake Setup: Glue Iceberg REST + Vended Credentials

## Purpose of this phase

Phase 1 created the Iceberg table.

Phase 2 prepared AWS Glue, Lake Formation, S3, and IAM.

Phase 3 connects Snowflake to the AWS Glue Iceberg REST catalog and exposes the external Iceberg table through a Snowflake catalog-linked database.

The final architecture is:

```text
                         Snowflake
                            |
                            v
                 Catalog-linked database
                            |
                            v
              Glue Iceberg REST integration
                            |
                         SigV4
                            |
                            v
                SnowflakeGlueVendedRole
                       /           \
                      v             v
                AWS Glue      Lake Formation
                                  |
                                  | temporary
                                  | scoped S3 credentials
                                  v
                              Amazon S3
                            /            \
                    Iceberg metadata    Parquet data
```

The important setting is:

```text
ACCESS_DELEGATION_MODE = VENDED_CREDENTIALS
```

This means a Snowflake External Volume is **not required** for this pattern.

---

# Before you start

Phase 1 and Phase 2 must already be complete.

You need these values:

```text
AWS account ID
AWS Region
Glue database name
AWS IAM role ARN for SnowflakeGlueVendedRole
```

Replace all values inside `<...>` below.

Placeholder meanings:

```text
<AWS_ACCOUNT_ID>
    Your 12-digit AWS account ID.

<AWS_REGION>
    The same Region used for S3, Glue and Lake Formation.

<GLUE_DATABASE>
    The Glue database created in Phase 1.
    Default in this guide: iceberg_poc
```

---

# Step 1 — Use an administrative Snowflake role for the POC

Run in Snowsight:

```sql
USE ROLE ACCOUNTADMIN;
```

### What is this doing?

It switches to a role that has permission to create account-level integrations.

### Why do we need this?

A catalog integration is an account-level Snowflake object.

For production, use a custom administrative role with only the required privileges instead of relying on `ACCOUNTADMIN` for normal operations.

---

# Step 2 — Create the AWS Glue Iceberg REST catalog integration

Replace the placeholders before running this SQL.

```sql
CREATE CATALOG INTEGRATION GLUE_ICEBERG_REST_VENDED_INT
    CATALOG_SOURCE = ICEBERG_REST
    TABLE_FORMAT = ICEBERG
    CATALOG_NAMESPACE = '<GLUE_DATABASE>'
    REST_CONFIG = (
        CATALOG_URI = 'https://glue.<AWS_REGION>.amazonaws.com/iceberg'
        CATALOG_API_TYPE = AWS_GLUE
        CATALOG_NAME = '<AWS_ACCOUNT_ID>'
        ACCESS_DELEGATION_MODE = VENDED_CREDENTIALS
    )
    REST_AUTHENTICATION = (
        TYPE = SIGV4
        SIGV4_IAM_ROLE =
          'arn:aws:iam::<AWS_ACCOUNT_ID>:role/SnowflakeGlueVendedRole'
        SIGV4_SIGNING_REGION = '<AWS_REGION>'
    )
    ENABLED = TRUE;
```

For this POC, `<GLUE_DATABASE>` is normally:

```text
iceberg_poc
```

### What is this doing?

It tells Snowflake how to connect to the AWS Glue Iceberg REST endpoint.

### Why do we need it?

Snowflake needs a catalog integration to:

- Discover Iceberg namespaces and tables.
- Ask Glue for the latest Iceberg metadata.
- Authenticate to AWS by using SigV4.
- Request catalog-vended storage credentials.

### What does `CATALOG_NAME` mean here?

For `CATALOG_API_TYPE = AWS_GLUE`, Snowflake expects the AWS account ID.

### What does `VENDED_CREDENTIALS` mean?

Instead of configuring a permanent Snowflake External Volume for S3, Snowflake asks the external catalog/Lake Formation for temporary scoped storage credentials.

---

# Step 3 — Retrieve the Snowflake-generated AWS identity

Run:

```sql
DESC CATALOG INTEGRATION GLUE_ICEBERG_REST_VENDED_INT;
```

Find and copy these two values:

```text
GLUE_AWS_IAM_USER_ARN
GLUE_AWS_EXTERNAL_ID
```

### What are these?

`GLUE_AWS_IAM_USER_ARN`

```text
The AWS IAM identity Snowflake uses when it assumes your AWS role.
```

`GLUE_AWS_EXTERNAL_ID`

```text
A security value used in the AWS role trust relationship.
```

### Why do we need them?

AWS must explicitly trust the Snowflake IAM identity before Snowflake can assume `SnowflakeGlueVendedRole`.

---

# Step 4 — Required AWS callback: replace the temporary IAM trust policy

This is the only AWS action inside Phase 3.

Return briefly to AWS CloudShell.

Set the values copied from Snowflake:

```bash
# Paste the exact GLUE_AWS_IAM_USER_ARN returned by Snowflake.
export SNOWFLAKE_IAM_USER_ARN="<GLUE_AWS_IAM_USER_ARN_FROM_SNOWFLAKE>"

# Paste the exact GLUE_AWS_EXTERNAL_ID returned by Snowflake.
export SNOWFLAKE_EXTERNAL_ID="<GLUE_AWS_EXTERNAL_ID_FROM_SNOWFLAKE>"

# Role created in Phase 2.
export SNOWFLAKE_GLUE_ROLE="SnowflakeGlueVendedRole"
```

Create the final trust policy:

```bash
cat > /tmp/snowflake-final-trust.json <<EOF_JSON
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSnowflakeAssumeRole",
      "Effect": "Allow",
      "Principal": {
        "AWS": "${SNOWFLAKE_IAM_USER_ARN}"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "${SNOWFLAKE_EXTERNAL_ID}"
        }
      }
    }
  ]
}
EOF_JSON
```

Replace the temporary Phase 2 trust policy:

```bash
aws iam update-assume-role-policy \
  --role-name "$SNOWFLAKE_GLUE_ROLE" \
  --policy-document file:///tmp/snowflake-final-trust.json
```

Validate:

```bash
aws iam get-role \
  --role-name "$SNOWFLAKE_GLUE_ROLE" \
  --query 'Role.AssumeRolePolicyDocument'
```

### What is this doing?

It changes the AWS role from a temporary bootstrap trust to the real Snowflake trust relationship.

### Why do we need the external ID?

The external ID helps protect against the confused-deputy problem when a third party such as Snowflake assumes a role in your AWS account.

### Important

If you later drop and recreate the Snowflake catalog integration, Snowflake can generate a different external ID.

If that happens, update the AWS trust policy again.

---

# Step 5 — Ask Snowflake what diagnostic operations are available

Back in Snowflake:

```sql
SELECT SYSTEM$EXECUTE_CATALOG_OPERATION(
    'GLUE_ICEBERG_REST_VENDED_INT',
    'help'
);
```

### What is this doing?

It lists the diagnostic catalog operations supported by the integration.

### Why do we start with this?

It gives us an easy way to validate each layer before creating the catalog-linked database.

---

# Step 6 — Verify basic catalog connectivity

```sql
SELECT SYSTEM$EXECUTE_CATALOG_OPERATION(
    'GLUE_ICEBERG_REST_VENDED_INT',
    'verify'
);
```

### What is this checking?

It checks whether Snowflake can connect to and authenticate with the configured catalog.

### Why is this useful?

It separates a basic Glue connectivity problem from a later Lake Formation or S3 credential problem.

---

# Step 7 — List the Glue tables visible to Snowflake

```sql
SELECT SYSTEM$EXECUTE_CATALOG_OPERATION(
    'GLUE_ICEBERG_REST_VENDED_INT',
    'listTables',
    'iceberg_poc'
);
```

Expected to include:

```text
customer
```

### What is this doing?

Snowflake asks the remote Glue Iceberg REST catalog for tables in the namespace.

### Why is this important?

If `customer` appears, we know that:

```text
Snowflake -> IAM role -> Glue catalog
```

is working.

---

# Step 8 — Test vended credentials

```sql
SELECT SYSTEM$EXECUTE_CATALOG_OPERATION(
    'GLUE_ICEBERG_REST_VENDED_INT',
    'getVendedCredentials',
    'iceberg_poc',
    'customer'
);
```

Expected logical result:

```json
{
  "status": "Credentials retrieved successfully"
}
```

### What is this doing?

Snowflake asks the catalog/Lake Formation for scoped temporary credentials for the table.

### Does Snowflake display the actual AWS access keys?

No.

The function only reports whether Snowflake was able to retrieve the credentials.

### Why is this the key test?

It proves that the following security path is working:

```text
Snowflake
   |
   v
SnowflakeGlueVendedRole
   |
   v
Lake Formation permissions
   |
   v
Temporary scoped S3 credentials
```

If this fails with:

```text
Forbidden: Access is not allowed
```

check Phase 2 again, especially:

```text
Lake Formation VerificationStatus = VERIFIED
AllowFullTableExternalDataAccess = true
Glue table is registered with Lake Formation
Snowflake role has Lake Formation SELECT
Snowflake role has lakeformation:GetDataAccess
Final IAM trust policy has the correct Snowflake ARN
Final IAM trust policy has the correct external ID
```

---

# Step 9 — Ask Glue for the current Iceberg metadata file

```sql
SELECT SYSTEM$EXECUTE_CATALOG_OPERATION(
    'GLUE_ICEBERG_REST_VENDED_INT',
    'loadTable',
    'iceberg_poc',
    'customer'
);
```

Expected logical output:

```json
{
  "metadataFile": "s3://<YOUR_BUCKET>/iceberg/customer/metadata/<CURRENT_METADATA_FILE>.metadata.json"
}
```

### What is this doing?

Snowflake asks the remote catalog for the latest Iceberg table metadata.

### Why is this useful?

It demonstrates an important Iceberg concept:

Glue is the catalog, but the detailed Iceberg metadata files still live in S3.

---

# Step 10 — Create a catalog-linked database

```sql
CREATE DATABASE ICEBERG_GLUE_VENDED_DB
    LINKED_CATALOG = (
        CATALOG = 'GLUE_ICEBERG_REST_VENDED_INT'
        ALLOWED_NAMESPACES = ('iceberg_poc')
    );
```

### What is this doing?

It creates a Snowflake database that is linked to the remote Iceberg REST catalog.

### Why is this powerful?

Snowflake can automatically discover namespaces and Iceberg tables from the external catalog instead of requiring us to manually create a Snowflake table definition for every remote table.

### Why is there no `EXTERNAL_VOLUME`?

Because this POC uses:

```text
ACCESS_DELEGATION_MODE = VENDED_CREDENTIALS
```

Lake Formation provides the temporary storage credentials.

---

# Step 11 — Check the namespaces discovered by Snowflake

```sql
SHOW SCHEMAS
IN DATABASE ICEBERG_GLUE_VENDED_DB;
```

### What should you see?

A schema corresponding to the Glue namespace/database:

```text
iceberg_poc
```

### Why do we check this?

It proves that the catalog-linked database is discovering the remote namespace.

---

# Step 12 — Check the discovered Iceberg tables

```sql
SHOW ICEBERG TABLES
IN DATABASE ICEBERG_GLUE_VENDED_DB;
```

Expected table:

```text
customer
```

### What is this doing?

It shows the remote Iceberg tables that Snowflake has discovered through the catalog-linked database.

---

# Step 13 — Query the external Iceberg table from Snowflake

AWS Glue object names are commonly lowercase. Quoting the identifiers preserves their exact case.

```sql
SELECT *
FROM ICEBERG_GLUE_VENDED_DB."iceberg_poc"."customer"
ORDER BY "customer_id";
```

Expected logical result:

```text
1 | Customer A | Agra   | 15000.00
2 | Customer B | Delhi  | 25000.00
3 | Customer C | Mumbai | 18000.00
```

### What is this doing?

Snowflake is querying the Iceberg table whose files are physically stored in Amazon S3.

### Is Snowflake copying these rows into normal Snowflake storage first?

No.

The table remains externally managed.

Snowflake uses the external catalog metadata and the temporary storage credentials to access the Iceberg table.

---

# Step 14 — Describe the external Iceberg table

```sql
DESC ICEBERG TABLE
ICEBERG_GLUE_VENDED_DB."iceberg_poc"."customer";
```

### Why do this?

It lets you inspect the Snowflake representation of the externally managed Iceberg table.

---

# Final validation checklist

At the end of Phase 3:

```text
Snowflake catalog integration exists                  ✅
Final AWS trust relationship is configured            ✅
Snowflake can verify the catalog                       ✅
Snowflake can list the Glue table                      ✅
Snowflake can retrieve vended credentials              ✅
Snowflake can resolve the current metadata.json        ✅
Catalog-linked database exists                         ✅
Glue namespace is visible in Snowflake                 ✅
Iceberg customer table is visible                      ✅
Snowflake SELECT returns the same three sample rows    ✅
```

---

# What the complete POC demonstrates

The completed architecture separates responsibilities cleanly:

```text
AWS Glue
= Catalog and table discovery

Amazon S3
= Physical Iceberg metadata and Parquet data

AWS Lake Formation
= Table authorization and temporary credential vending

Snowflake catalog integration
= Connection to the Glue Iceberg REST endpoint

Snowflake catalog-linked database
= Automatically exposes the remote catalog in Snowflake
```

The most important architectural point is:

> Snowflake can discover and query an externally managed Iceberg table without moving the underlying table into Snowflake-managed storage.

This is the practical foundation for using Snowflake Horizon Catalog as part of a broader metadata-hub and open-data architecture.
