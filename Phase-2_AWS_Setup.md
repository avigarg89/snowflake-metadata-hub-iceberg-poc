# Phase 2 — AWS Setup: Lake Formation and Vended Credentials

## Purpose of this phase

Phase 1 created the Iceberg table.

Phase 2 prepares AWS so that Snowflake can securely discover and read that table through the AWS Glue Iceberg REST catalog.

The target AWS-side architecture is:

```text
Snowflake
   |
   | later assumes an AWS IAM role
   v
SnowflakeGlueVendedRole
   |
   +----> AWS Glue Iceberg REST
   |
   +----> AWS Lake Formation
                |
                | validates table permissions
                | and vends temporary credentials
                v
      LakeFormationIcebergDataRole
                |
                v
             Amazon S3
```

There are **two IAM roles** because they have different jobs:

```text
SnowflakeGlueVendedRole
= Snowflake -> Glue / Lake Formation

LakeFormationIcebergDataRole
= Lake Formation -> S3
```

Keeping these responsibilities separate makes the security model easier to understand.

---

# Before you start

Phase 1 must already be complete.

You should already have:

```text
S3 bucket
Glue database: iceberg_poc
Glue table:    customer
```

You also need permissions to manage:

- IAM roles and policies
- AWS Lake Formation
- AWS Glue

---

# Step 1 — Set reusable values

Replace the values inside `<...>`.

```bash
# Use the same Region that was used in Phase 1.
export AWS_REGION="<YOUR_AWS_REGION>"

# Use the same S3 bucket created in Phase 1.
export S3_BUCKET="<YOUR_UNIQUE_S3_BUCKET_NAME>"

# Normally keep these names unchanged for this POC.
export GLUE_DATABASE="iceberg_poc"
export ICEBERG_TABLE="customer"

# AWS account ID is detected automatically.
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Role used internally by Lake Formation to access S3.
export LF_DATA_ROLE="LakeFormationIcebergDataRole"

# Role that Snowflake will later assume to call Glue/Lake Formation.
export SNOWFLAKE_GLUE_ROLE="SnowflakeGlueVendedRole"

# We register the parent Iceberg prefix so multiple Iceberg tables can live below it.
export LF_RESOURCE_ARN="arn:aws:s3:::${S3_BUCKET}/iceberg"

echo "AWS_ACCOUNT_ID      = $AWS_ACCOUNT_ID"
echo "AWS_REGION          = $AWS_REGION"
echo "S3_BUCKET           = $S3_BUCKET"
echo "LF_RESOURCE_ARN     = $LF_RESOURCE_ARN"
```

### Why are we registering `/iceberg` instead of `/iceberg/customer`?

It gives Lake Formation control over the Iceberg area of the bucket rather than only one table folder.

That makes the pattern reusable if more Iceberg tables are added later.

---

# Step 2 — Create the Lake Formation S3 access role

Create the trust policy:

```bash
cat > /tmp/lakeformation-trust.json <<'JSON'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lakeformation.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
JSON
```

Create the role:

```bash
aws iam create-role \
  --role-name "$LF_DATA_ROLE" \
  --assume-role-policy-document file:///tmp/lakeformation-trust.json
```

### What is this doing?

It creates an IAM role that the Lake Formation service is allowed to assume.

### Why do we need this?

Lake Formation needs a secure way to access the S3 files that belong to the registered data location.

This role is **not** the role Snowflake assumes.

---

# Step 3 — Give the Lake Formation role access to the Iceberg S3 prefix

Create the policy document:

```bash
cat > /tmp/lakeformation-s3-policy.json <<EOF_JSON
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "IcebergObjectAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::${S3_BUCKET}/iceberg/*"
      ]
    },
    {
      "Sid": "IcebergBucketList",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::${S3_BUCKET}"
      ],
      "Condition": {
        "StringLike": {
          "s3:prefix": [
            "iceberg",
            "iceberg/*"
          ]
        }
      }
    }
  ]
}
EOF_JSON
```

Attach the policy as an inline role policy:

```bash
aws iam put-role-policy \
  --role-name "$LF_DATA_ROLE" \
  --policy-name LakeFormationIcebergS3Access \
  --policy-document file:///tmp/lakeformation-s3-policy.json
```

### What is this doing?

It lets Lake Formation access objects only under the Iceberg area of this bucket.

### Why do we need this?

Lake Formation cannot vend usable temporary access to the registered location unless its registration role can access that location.

### Why use an inline policy here?

For a small POC, an inline policy is simple and avoids managing versions of a separate customer-managed IAM policy.

---

# Step 4 — Register the Iceberg S3 location with Lake Formation

```bash
aws lakeformation register-resource \
  --region "$AWS_REGION" \
  --resource-arn "$LF_RESOURCE_ARN" \
  --role-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:role/${LF_DATA_ROLE}" \
  --expected-resource-owner-account "$AWS_ACCOUNT_ID"
```

### What is this doing?

It tells Lake Formation:

> This S3 prefix contains data that Lake Formation should manage.

### Why is `--expected-resource-owner-account` important?

It allows Lake Formation to verify that the registration role can really access the S3 location.

It is also important for the temporary-credential-vending pattern.

---

# Step 5 — Critical check: confirm the registration is VERIFIED

```bash
aws lakeformation describe-resource \
  --region "$AWS_REGION" \
  --resource-arn "$LF_RESOURCE_ARN"
```

Look for:

```text
VerificationStatus = VERIFIED
```

### What does VERIFIED mean?

It means Lake Formation confirmed that the registered IAM role has sufficient permissions to access the S3 location.

### Why is this the most important AWS checkpoint?

If the status is:

```text
NOT_VERIFIED
```

or:

```text
VERIFICATION_FAILED
```

do **not** continue.

A bad registration can later produce confusing errors such as:

```text
Forbidden: Access is not allowed
```

when Snowflake asks for vended credentials.

---

# Step 6 — Confirm the Glue table is now registered with Lake Formation

```bash
aws glue get-table \
  --region "$AWS_REGION" \
  --database-name "$GLUE_DATABASE" \
  --name "$ICEBERG_TABLE" \
  --query 'Table.{Name:Name,Location:StorageDescriptor.Location,RegisteredWithLakeFormation:IsRegisteredWithLakeFormation}'
```

Expected:

```text
RegisteredWithLakeFormation = true
```

### What is this doing?

It confirms that the Glue table points to a location managed by Lake Formation.

### Why does this matter?

Lake Formation can only enforce its table permissions and vend scoped credentials when it recognizes the table's S3 location as registered.

---

# Step 7 — Allow full-table credential vending for external engines

First retrieve the existing Lake Formation settings.

```bash
aws lakeformation get-data-lake-settings \
  --region "$AWS_REGION" \
  --query 'DataLakeSettings' \
  > /tmp/lakeformation-settings.json
```

Update only the setting we need:

```bash
jq '.AllowFullTableExternalDataAccess = true' \
  /tmp/lakeformation-settings.json \
  > /tmp/lakeformation-settings-updated.json
```

Apply the settings:

```bash
aws lakeformation put-data-lake-settings \
  --region "$AWS_REGION" \
  --data-lake-settings file:///tmp/lakeformation-settings-updated.json
```

Validate:

```bash
aws lakeformation get-data-lake-settings \
  --region "$AWS_REGION" \
  --query 'DataLakeSettings.AllowFullTableExternalDataAccess'
```

Expected:

```text
true
```

### What is this doing?

It allows an external engine such as Snowflake to receive temporary storage credentials when it has full-table Lake Formation access.

### Why did we first download the existing settings?

`put-data-lake-settings` updates the complete settings object.

By reading the current settings first and changing only one property, we avoid accidentally overwriting existing Lake Formation administrators or other account settings.

---

# Step 8 — Create the IAM role that Snowflake will later assume

Snowflake has not generated its AWS IAM user ARN and external ID yet. Those values appear only after the Snowflake catalog integration is created in Phase 3.

For now we create the role with a **temporary bootstrap trust policy**.

```bash
cat > /tmp/snowflake-glue-bootstrap-trust.json <<EOF_JSON
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "TemporaryBootstrapTrust",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::${AWS_ACCOUNT_ID}:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "REPLACE-AFTER-SNOWFLAKE-INTEGRATION"
        }
      }
    }
  ]
}
EOF_JSON
```

Create the role:

```bash
aws iam create-role \
  --role-name "$SNOWFLAKE_GLUE_ROLE" \
  --assume-role-policy-document file:///tmp/snowflake-glue-bootstrap-trust.json
```

### What is this doing?

It creates the IAM role whose ARN we will give to Snowflake.

### Why is the trust policy temporary?

We do not yet know Snowflake's generated IAM user ARN or external ID.

In Phase 3, we replace this entire trust policy with Snowflake's exact identity.

### Security note

Do not leave the bootstrap trust policy in place.

The final policy will trust only Snowflake's generated AWS IAM user and external ID.

---

# Step 9 — Give the Snowflake role read access to the Glue catalog

Create the policy:

```bash
cat > /tmp/snowflake-glue-access-policy.json <<EOF_JSON
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "GlueIcebergCatalogRead",
      "Effect": "Allow",
      "Action": [
        "glue:GetCatalog",
        "glue:GetDatabase",
        "glue:GetDatabases",
        "glue:GetTable",
        "glue:GetTables"
      ],
      "Resource": [
        "arn:aws:glue:${AWS_REGION}:${AWS_ACCOUNT_ID}:catalog",
        "arn:aws:glue:${AWS_REGION}:${AWS_ACCOUNT_ID}:database/${GLUE_DATABASE}",
        "arn:aws:glue:${AWS_REGION}:${AWS_ACCOUNT_ID}:table/${GLUE_DATABASE}/*"
      ]
    },
    {
      "Sid": "LakeFormationCredentialAccess",
      "Effect": "Allow",
      "Action": [
        "lakeformation:GetDataAccess"
      ],
      "Resource": "*"
    }
  ]
}
EOF_JSON
```

Attach it:

```bash
aws iam put-role-policy \
  --role-name "$SNOWFLAKE_GLUE_ROLE" \
  --policy-name SnowflakeGlueVendedAccess \
  --policy-document file:///tmp/snowflake-glue-access-policy.json
```

### What is this doing?

This role can now:

```text
Read Glue catalog metadata
Ask Lake Formation for scoped data access
```

### Why is there no permanent S3 `GetObject` permission on this role?

Because this design uses **vended credentials**.

Snowflake does not need a permanent S3 role for table-file access. Lake Formation will provide temporary scoped storage credentials.

---

# Step 10 — Grant Lake Formation database permission

```bash
aws lakeformation grant-permissions \
  --region "$AWS_REGION" \
  --principal DataLakePrincipalIdentifier="arn:aws:iam::${AWS_ACCOUNT_ID}:role/${SNOWFLAKE_GLUE_ROLE}" \
  --resource "{\"Database\":{\"CatalogId\":\"${AWS_ACCOUNT_ID}\",\"Name\":\"${GLUE_DATABASE}\"}}" \
  --permissions DESCRIBE
```

### What is this doing?

It allows the Snowflake role to see the Glue/Lake Formation database.

### Why do we need this?

Without database visibility, table discovery can fail even if the role has IAM Glue API permissions.

---

# Step 11 — Grant Lake Formation access to the Iceberg table

```bash
aws lakeformation grant-permissions \
  --region "$AWS_REGION" \
  --principal DataLakePrincipalIdentifier="arn:aws:iam::${AWS_ACCOUNT_ID}:role/${SNOWFLAKE_GLUE_ROLE}" \
  --resource "{\"Table\":{\"CatalogId\":\"${AWS_ACCOUNT_ID}\",\"DatabaseName\":\"${GLUE_DATABASE}\",\"Name\":\"${ICEBERG_TABLE}\"}}" \
  --permissions SELECT DESCRIBE
```

### What is this doing?

It gives the Snowflake role permission to read the complete Iceberg table.

### Why do we need both IAM and Lake Formation permissions?

They solve different problems:

```text
IAM permissions
= Can the role call the AWS Glue / Lake Formation APIs?

Lake Formation permissions
= Is the role allowed to see and read this specific governed table?
```

Both layers must allow the request.

---

# Step 12 — Validate the Lake Formation grants

Database permission:

```bash
aws lakeformation list-permissions \
  --region "$AWS_REGION" \
  --principal DataLakePrincipalIdentifier="arn:aws:iam::${AWS_ACCOUNT_ID}:role/${SNOWFLAKE_GLUE_ROLE}" \
  --resource "{\"Database\":{\"CatalogId\":\"${AWS_ACCOUNT_ID}\",\"Name\":\"${GLUE_DATABASE}\"}}"
```

Table permission:

```bash
aws lakeformation list-permissions \
  --region "$AWS_REGION" \
  --principal DataLakePrincipalIdentifier="arn:aws:iam::${AWS_ACCOUNT_ID}:role/${SNOWFLAKE_GLUE_ROLE}" \
  --resource "{\"Table\":{\"CatalogId\":\"${AWS_ACCOUNT_ID}\",\"DatabaseName\":\"${GLUE_DATABASE}\",\"Name\":\"${ICEBERG_TABLE}\"}}"
```

### What should you see?

Conceptually:

```text
Database:
DESCRIBE

Table:
SELECT
DESCRIBE
```

Lake Formation can sometimes display a full-column SELECT grant internally as a `TableWithColumns` resource with a `ColumnWildcard`. That still represents access to all columns.

---

# Step 13 — Record the Snowflake IAM role ARN

```bash
echo "Use this role ARN in Phase 3:"
echo "arn:aws:iam::${AWS_ACCOUNT_ID}:role/${SNOWFLAKE_GLUE_ROLE}"
```

### Why do we need this?

The Snowflake catalog integration needs this IAM role ARN for SigV4 authentication.

---

# Phase 2 checkpoint

Do not start the Snowflake configuration until all of these are true:

```text
LakeFormationIcebergDataRole exists                  ✅
S3 access policy is attached                         ✅
S3 /iceberg location is registered                   ✅
VerificationStatus = VERIFIED                        ✅
Glue customer table reports LF registered            ✅
AllowFullTableExternalDataAccess = true              ✅
SnowflakeGlueVendedRole exists                       ✅
Glue/Lake Formation IAM policy is attached           ✅
Lake Formation DESCRIBE permission is granted        ✅
Lake Formation SELECT permission is granted          ✅
```

At this point AWS is ready.

The only unfinished AWS item is the **final trust policy** on `SnowflakeGlueVendedRole`.

That cannot be completed yet because Snowflake generates the required IAM user ARN and external ID during Phase 3.
