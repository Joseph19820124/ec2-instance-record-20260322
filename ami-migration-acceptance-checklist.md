# AMI Migration n8n Acceptance Checklist

This checklist is for validating the event-driven n8n orchestration after triggering the flow from Step 1.

## Scope

The current chain is:

`Step 1 -> Step 2 -> Step 3 -> Step 4 -> Step 5 -> Step 7 -> Step 8`

Notes:

- `Step 6` is preparatory only and is not modeled as a queue job.
- `Step 3` should create or reuse the deterministic AWS S3 bucket:
  `joseph-aws-gcp-ami-{aws-account-id}`
- `Step 4` should create or reuse the AWS `vmimport` role.

## Step 3: Check or Create Export Bucket

Expected result:

- The callback succeeds.
- The bucket name is `joseph-aws-gcp-ami-{12-digit-account-id}`.
- If the bucket did not exist before the run:
  - `bucketExists = false`
  - `bucketCreated = true`
- If the bucket already existed:
  - `bucketExists = true`
  - `bucketCreated = false`

Validation:

```bash
aws s3api list-buckets --query "Buckets[?starts_with(Name, 'joseph-aws-gcp-ami-')].Name"
```

Check in n8n:

- `AMI Migration - Step 3 Callback Receiver`
- confirm the latest execution contains:
  - `s3Bucket`
  - `bucketExists`
  - `bucketCreated`

## Step 4: Ensure vmimport Role

Expected result:

- The callback succeeds.
- The role name is `vmimport`.
- If the role did not exist before the run:
  - `roleCreated = true`
- If the role already existed:
  - `roleCreated = false`

Validation:

```bash
aws iam get-role --role-name vmimport
aws iam get-role-policy --role-name vmimport --policy-name vmimport
```

Check in n8n:

- `AMI Migration - Step 4 Callback Receiver`
- confirm the latest execution contains:
  - `roleName`
  - `roleCreated`
  - `policyName`
  - `s3Bucket`

## Step 5: Export AMI to S3

Expected result:

- The export task is created in AWS.
- The task eventually reaches `completed`.
- The exported `.vmdk` appears in the Step 3 bucket under `exports/`.
- `Step 5 Callback` succeeds with:
  - `ok = true`
  - `exportImageTaskId`
  - `status = completed`

Validation while running:

```bash
aws ec2 describe-export-image-tasks
aws s3 ls s3://joseph-aws-gcp-ami-{account-id}/exports/
```

Check in n8n:

- `AMI Migration - Step 5 Callback Receiver`
- confirm the latest execution contains:
  - `exportImageTaskId`
  - `status`
  - `s3Bucket`
  - `s3Key` if present

## Step 7: Copy Exported Image from S3 to GCS

Expected result:

- `Step 5 Callback` auto-enqueues `Step 7`.
- The `.vmdk` is copied to the configured GCS bucket.
- `Step 7 Callback` succeeds.

Validation:

```bash
gcloud storage ls gs://<gcs-bucket>/
gcloud storage ls -L gs://<gcs-bucket>/<object-name>
```

Check in n8n:

- `AMI Migration - Step 7 Callback Receiver`
- confirm the latest execution contains:
  - `sourceS3Bucket`
  - `sourceS3Key`
  - `gcsBucket`
  - `gcsObject`

## Step 8: Import GCP Image

Expected result:

- `Step 7 Callback` auto-enqueues `Step 8`.
- A new GCP custom image is created.
- `Step 8 Callback` succeeds.

Validation:

```bash
gcloud compute images list --filter="name~ami-migration"
gcloud compute images describe <image-name>
```

Check in n8n:

- `AMI Migration - Step 8 Callback Receiver`
- confirm the latest execution contains:
  - `imageName`
  - `imageFamily`
  - `gcpProject`
  - final import status

## Minimum Pass Criteria

The run is accepted when all of the following are true:

- Step 3 created or reused the expected deterministic S3 bucket.
- Step 4 created or reused the `vmimport` role.
- Step 5 produced a completed AWS export task and a `.vmdk` in S3.
- Step 7 copied the `.vmdk` into the target GCS bucket.
- Step 8 created the target GCP image successfully.

## Current Operational Note

`Step 5` can take a long time. A real export may stay in `active` for 40 to 50 minutes before the callback returns.
