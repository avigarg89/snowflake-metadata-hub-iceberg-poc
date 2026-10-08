# Phase 1 — AWS Setup: Create the Iceberg Table

## Purpose of this phase

In this phase we create the basic AWS Iceberg environment.

By the end of this phase, we will have:

- An Amazon S3 bucket that stores the Iceberg files.
- An AWS Glue database that acts as the catalog.
- An Apache Iceberg table called `customer`.
- A few sample records inserted through Amazon Athena.
- A working Iceberg table that exists completely outside Snowflake.

The architecture at the end of Phase 1 is:

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
  └── iceberg/customer/
       ├── metadata/
       └── data/
```

---

# Before you start

Run the commands in **AWS CloudShell** or another terminal where the AWS CLI is already authenticated.

You need permissions to create and manage:

- S3
- AWS Glue
- Athena

---

# Step 1 — Set reusable values

Update the values inside `<...>` before running the commands.

```bash
# AWS Region where you want to build the POC.
# Example: us-west-2, eu-west-1, ap-southeast-2
export AWS_REGION="<YOUR_AWS_REGION>"

# S3 bucket names must be globally unique.
# Example: mycompany-snowflake-iceberg-poc
export S3_BUCKET="<YOUR_UNIQUE_S3_BUCKET_NAME>"

# These names can normally be kept as-is for this POC.
export GLUE_DATABASE="iceberg_poc"
export ICEBERG_TABLE="customer"

# Automatically retrieve the AWS account ID of the logged-in AWS identity.
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Physical location of the Iceberg table.
export ICEBERG_PATH="s3://${S3_BUCKET}/iceberg/customer/"

# Athena needs an S3 location for query-result files.
export ATHENA_OUTPUT="s3://${S3_BUCKET}/athena-results/"

echo "AWS_ACCOUNT_ID = $AWS_ACCOUNT_ID"
echo "AWS_REGION     = $AWS_REGION"
echo "S3_BUCKET      = $S3_BUCKET"
echo "ICEBERG_PATH   = $ICEBERG_PATH"
```

### What is this doing?

Instead of writing the account ID, Region, bucket name, and table name again and again, we store them in variables once.

### Why do we need this?

It keeps the commands readable and makes the guide reusable for another AWS account.

---

# Step 2 — Confirm which AWS identity is being used

```bash
aws sts get-caller-identity
```

### What is this doing?

It shows the AWS account and IAM identity currently being used by the CLI.

### Why do we need this?

Before creating resources, it is useful to confirm that you are working in the intended AWS account.

---

# Step 3 — Create the S3 bucket

For `us-east-1`, AWS uses a slightly different bucket-creation command. The conditional below handles both cases.

```bash
if [ "$AWS_REGION" = "us-east-1" ]; then
  aws s3api create-bucket \
    --bucket "$S3_BUCKET" \
    --region "$AWS_REGION"
else
  aws s3api create-bucket \
    --bucket "$S3_BUCKET" \
    --region "$AWS_REGION" \
    --create-bucket-configuration LocationConstraint="$AWS_REGION"
fi
```

### What is this doing?

It creates an S3 bucket.

### Why do we need this?

S3 is the physical storage layer for the Iceberg table.

Later, this bucket will contain files similar to:

```text
iceberg/customer/
├── metadata/
│   ├── *.metadata.json
│   └── *.avro
└── data/
    └── *.parquet
```

The Parquet files contain the actual table data. The metadata files describe Iceberg snapshots, schemas, manifests, and table state.

---

# Step 4 — Block public access to the bucket

```bash
aws s3api put-public-access-block \
  --bucket "$S3_BUCKET" \
  --public-access-block-configuration '{
    "BlockPublicAcls": true,
    "IgnorePublicAcls": true,
    "BlockPublicPolicy": true,
    "RestrictPublicBuckets": true
  }'
```

### What is this doing?

It prevents the bucket from accidentally becoming publicly accessible.

### Why do we need this?

Snowflake, Athena, Glue, and Lake Formation will use IAM-based access. Public S3 access is not required.

---

# Step 5 — Enable default S3 encryption

```bash
aws s3api put-bucket-encryption \
  --bucket "$S3_BUCKET" \
  --server-side-encryption-configuration '{
    "Rules": [
      {
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "AES256"
        }
      }
    ]
  }'
```

### What is this doing?

It enables S3-managed server-side encryption.

### Why do we need this?

The files are encrypted at rest without introducing the extra KMS permissions that would be required with a customer-managed KMS key.

For a production implementation, your organization might prefer SSE-KMS.

---

# Step 6 — Confirm the bucket exists

```bash
aws s3api head-bucket \
  --bucket "$S3_BUCKET"

echo "Bucket is accessible."
```

### What is this doing?

It verifies that the bucket exists and that the current AWS identity can access it.

### Why do we need this?

It is better to detect an S3 problem now instead of after creating the Iceberg table.

---

# Step 7 — Create the AWS Glue database

```bash
aws glue create-database \
  --region "$AWS_REGION" \
  --database-input '{
    "Name": "iceberg_poc",
    "Description": "Apache Iceberg POC for Snowflake metadata hub"
  }'
```

### What is this doing?

It creates a Glue database named:

```text
iceberg_poc
```

### Why do we need this?

AWS Glue is acting as the catalog.

A simple way to think about the components is:

```text
Glue = knows what the table is and where it is
S3   = stores the actual Iceberg files
```

---

# Step 8 — Validate the Glue database

```bash
aws glue get-database \
  --region "$AWS_REGION" \
  --name "$GLUE_DATABASE" \
  --query 'Database.Name' \
  --output text
```

Expected result:

```text
iceberg_poc
```

### Why do we check this?

It confirms that the catalog layer is ready before we create the table.

---

# Step 9 — Create a small Athena helper function

Athena queries run asynchronously. This helper starts a query and waits until it finishes.

```bash
athena_run() {

  local SQL="$1"

  LAST_QUERY_ID=$(aws athena start-query-execution \
    --region "$AWS_REGION" \
    --work-group primary \
    --query-string "$SQL" \
    --result-configuration OutputLocation="$ATHENA_OUTPUT" \
    --query 'QueryExecutionId' \
    --output text)

  echo "QueryExecutionId: $LAST_QUERY_ID"

  while true
  do
    STATE=$(aws athena get-query-execution \
      --region "$AWS_REGION" \
      --query-execution-id "$LAST_QUERY_ID" \
      --query 'QueryExecution.Status.State' \
      --output text)

    if [ "$STATE" = "SUCCEEDED" ]; then
      echo "Athena query succeeded."
      break
    fi

    if [ "$STATE" = "FAILED" ] || [ "$STATE" = "CANCELLED" ]; then
      echo "Athena query ended with state: $STATE"

      aws athena get-query-execution \
        --region "$AWS_REGION" \
        --query-execution-id "$LAST_QUERY_ID" \
        --query 'QueryExecution.Status.StateChangeReason' \
        --output text

      return 1
    fi

    sleep 2
  done
}
```

### What is this doing?

It hides the repetitive polling required by Athena.

### Why do we need this?

After every Athena command, we want to know whether it really succeeded before moving to the next step.

---

# Step 10 — Create the Iceberg table

```bash
athena_run "
CREATE TABLE ${GLUE_DATABASE}.${ICEBERG_TABLE} (
    customer_id   BIGINT,
    customer_name STRING,
    city          STRING,
    balance       DECIMAL(18,2)
)
LOCATION '${ICEBERG_PATH}'
TBLPROPERTIES (
    'table_type'='ICEBERG',
    'format'='parquet'
)
"
```

### What is this doing?

Athena creates an Apache Iceberg table and registers it in AWS Glue.

### Why do we need this?

This gives us a real external Iceberg table that Snowflake can discover later.

The table is cataloged as:

```text
iceberg_poc.customer
```

while its physical files are stored under:

```text
s3://<YOUR_BUCKET>/iceberg/customer/
```

### Important note

For Athena DDL, use:

```text
STRING
```

rather than plain `VARCHAR`.

---

# Step 11 — Insert sample records

```bash
athena_run "
INSERT INTO ${GLUE_DATABASE}.${ICEBERG_TABLE}
VALUES
    (1, 'Customer A', 'Agra',   DECIMAL '15000.00'),
    (2, 'Customer B', 'Delhi',  DECIMAL '25000.00'),
    (3, 'Customer C', 'Mumbai', DECIMAL '18000.00')
"
```

### What is this doing?

It writes three rows into the Iceberg table.

### What happens behind the scenes?

Iceberg creates new physical and metadata files.

Conceptually:

```text
INSERT
   |
   v
New Parquet data file
   |
   v
New Iceberg manifest information
   |
   v
New metadata.json
   |
   v
New table snapshot
```

### Why is this useful?

It gives us enough data to prove later that Snowflake is querying the same external table rather than a copied Snowflake table.

---

# Step 12 — Query the Iceberg table in Athena

```bash
athena_run "
SELECT *
FROM ${GLUE_DATABASE}.${ICEBERG_TABLE}
ORDER BY customer_id
"
```

Display the result:

```bash
aws athena get-query-results \
  --region "$AWS_REGION" \
  --query-execution-id "$LAST_QUERY_ID"
```

Expected logical result:

```text
1 | Customer A | Agra   | 15000.00
2 | Customer B | Delhi  | 25000.00
3 | Customer C | Mumbai | 18000.00
```

### Why do we do this?

Before involving Snowflake, we first prove that the Iceberg table itself is healthy.

---

# Step 13 — Validate the Glue table

```bash
aws glue get-table \
  --region "$AWS_REGION" \
  --database-name "$GLUE_DATABASE" \
  --name "$ICEBERG_TABLE" \
  --query 'Table.{Name:Name,Location:StorageDescriptor.Location,Parameters:Parameters}'
```

### What is this doing?

It confirms that Glue knows about the table.

### Why does this matter?

Snowflake will later connect to the **Glue Iceberg REST catalog**, so the Glue catalog entry must exist.

---

# Step 14 — Look at the physical Iceberg files in S3

```bash
aws s3 ls \
  "s3://${S3_BUCKET}/iceberg/customer/" \
  --recursive
```

You should see files under locations similar to:

```text
iceberg/customer/data/
iceberg/customer/metadata/
```

### What are these files?

```text
*.parquet
    Actual table data

*.metadata.json
    Iceberg table metadata and snapshot state

*.avro
    Iceberg manifest and manifest-list information
```

---

# Phase 1 checkpoint

Do not move to Phase 2 until all of these are true:

```text
S3 bucket exists                         ✅
Bucket is private                        ✅
S3 encryption is enabled                 ✅
Glue database exists                     ✅
Iceberg table exists                     ✅
Athena SELECT returns the sample rows    ✅
Iceberg files are visible in S3          ✅
```

At this point:

```text
Glue knows about the table
        +
S3 contains the table
        +
Athena can query the table
```

Snowflake has not been configured yet. That is intentional.

Phase 2 adds AWS Lake Formation and the IAM roles required for secure credential vending.
