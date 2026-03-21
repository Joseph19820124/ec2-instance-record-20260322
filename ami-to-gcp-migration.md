# AMI to GCP Migration

This document captures the officially supported migration path for moving an AWS AMI into Google Cloud as a custom image:

`AMI -> S3 -> GCS -> GCP custom image`

## Overview

AWS does not support exporting an AMI directly to Google Cloud Storage. The supported path is:

1. Export the AMI from AWS to an S3 bucket using VM Import/Export.
2. Copy the exported disk image from S3 to Google Cloud Storage.
3. Import that disk image into Google Cloud as a custom image.

References:

- AWS VM Import/Export: https://docs.aws.amazon.com/vm-import/latest/userguide/vmexport_image.html
- AWS export limitations: https://docs.aws.amazon.com/vm-import/latest/userguide/limits-image-export.html
- GCP image import: https://cloud.google.com/compute/docs/import/import-existing-image

## Prerequisites

- The AMI must be exportable.
- The AMI must not be blocked by AWS Marketplace or other export restrictions.
- You need an AWS S3 bucket in the same AWS account used for export.
- You need a GCP project and a GCS bucket.
- You need `aws`, `gcloud`, and optionally `gsutil` installed locally.

## Variables

Set these before running the commands:

```bash
export AWS_PROFILE=aws-4
export AWS_REGION=us-west-1
export AMI_ID=ami-00dabab179635c560
export S3_BUCKET=your-export-bucket
export S3_PREFIX=exports/

export GCP_PROJECT=your-gcp-project
export GCS_BUCKET=your-gcs-bucket
export GCP_ZONE=us-west1-a
export IMAGE_NAME=imported-ami-image
```

## Step 1: Verify the AMI in AWS

```bash
aws ec2 describe-images \
  --profile "$AWS_PROFILE" \
  --region "$AWS_REGION" \
  --image-ids "$AMI_ID"
```

## Step 2: Create the AWS `vmimport` Role

If the account does not already have the `vmimport` role, create it.

Create `trust-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "vmie.amazonaws.com" },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": { "sts:Externalid": "vmimport" }
      }
    }
  ]
}
```

Create the role:

```bash
aws iam create-role \
  --profile "$AWS_PROFILE" \
  --role-name vmimport \
  --assume-role-policy-document file://trust-policy.json
```

Create `role-policy.json` and replace `your-export-bucket` with the real bucket name:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketLocation",
        "s3:GetObject",
        "s3:ListBucket",
        "s3:PutObject",
        "s3:GetBucketAcl"
      ],
      "Resource": [
        "arn:aws:s3:::your-export-bucket",
        "arn:aws:s3:::your-export-bucket/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:ModifySnapshotAttribute",
        "ec2:CopySnapshot",
        "ec2:RegisterImage",
        "ec2:Describe*"
      ],
      "Resource": "*"
    }
  ]
}
```

Attach the inline policy:

```bash
aws iam put-role-policy \
  --profile "$AWS_PROFILE" \
  --role-name vmimport \
  --policy-name vmimport \
  --policy-document file://role-policy.json
```

## Step 3: Export the AMI to S3

Start the export task. `VMDK` is a common choice for GCP import workflows.

```bash
aws ec2 export-image \
  --profile "$AWS_PROFILE" \
  --region "$AWS_REGION" \
  --image-id "$AMI_ID" \
  --disk-image-format VMDK \
  --s3-export-location S3Bucket="$S3_BUCKET",S3Prefix="$S3_PREFIX"
```

Check progress:

```bash
aws ec2 describe-export-image-tasks \
  --profile "$AWS_PROFILE" \
  --region "$AWS_REGION"
```

When complete, the exported file will be in S3 under a path similar to:

```bash
s3://your-export-bucket/exports/export-ami-xxxx.vmdk
```

## Step 4: Prepare GCP

Authenticate and select the target project:

```bash
gcloud auth login
gcloud config set project "$GCP_PROJECT"
```

Create the destination GCS bucket:

```bash
gcloud storage buckets create "gs://$GCS_BUCKET" --location=us-west1
```

## Step 5: Copy the Exported Image from S3 to GCS

One simple approach is local relay:

```bash
mkdir -p ./exported-image

aws s3 cp "s3://$S3_BUCKET/$S3_PREFIX" ./exported-image/ \
  --recursive \
  --profile "$AWS_PROFILE"

gcloud storage cp ./exported-image/*.vmdk "gs://$GCS_BUCKET/"
```

If the image is large, consider a server-side transfer approach instead of pulling the file through your laptop.

## Step 6: Import the Disk Image into GCP

Import the file from GCS into Compute Engine as a custom image:

```bash
gcloud compute images import "$IMAGE_NAME" \
  --source-file="gs://$GCS_BUCKET/export-ami-xxxx.vmdk" \
  --os=debian-12 \
  --quiet
```

Replace `--os=debian-12` with the correct guest OS for the AMI. Common values include:

- `ubuntu-2204`
- `ubuntu-2004`
- `rhel-8`
- `centos-7`
- `windows-2019`

## Step 7: Launch a Test VM in GCP

```bash
gcloud compute instances create test-imported-vm \
  --zone="$GCP_ZONE" \
  --image="$IMAGE_NAME"
```

## Validation Checklist

- Confirm the export task completed successfully in AWS.
- Confirm the `.vmdk` file exists in GCS.
- Confirm the custom image exists in GCP.
- Boot a test VM from the image.
- Validate networking, disk mapping, startup behavior, and application health.

## Important Notes

- AWS cannot export an AMI directly to GCS.
- The AWS-supported export target is S3 in the same AWS account.
- Not every AMI can be exported.
- Imported VMs often need guest OS adjustments after landing in GCP, especially around drivers, network configuration, and boot behavior.
- Windows imports may require additional driver and licensing review.
