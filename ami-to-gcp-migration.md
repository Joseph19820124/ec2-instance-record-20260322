# AMI to GCP Migration

This document captures the officially supported migration path for moving a running EC2 instance in AWS into Google Cloud as a custom image:

`running EC2 -> AMI -> S3 -> GCS -> GCP custom image`

## Overview

AWS does not support exporting an EC2 instance or an AMI directly to Google Cloud Storage. The supported path is:

1. Scan the AWS California region (`us-west-1`) for running EC2 instances.
2. Create an AMI from the target running EC2 instance.
3. Check for an account-unique S3 bucket and create it if it does not already exist.
4. Export the AMI to that S3 bucket using VM Import/Export.
5. Copy the exported disk image from S3 to Google Cloud Storage.
6. Import that disk image into Google Cloud as a custom image.

References:

- AWS VM Import/Export: https://docs.aws.amazon.com/vm-import/latest/userguide/vmexport_image.html
- AWS export limitations: https://docs.aws.amazon.com/vm-import/latest/userguide/limits-image-export.html
- GCP image import: https://cloud.google.com/compute/docs/import/import-existing-image

## Prerequisites

- The AMI must be exportable.
- The AMI must not be blocked by AWS Marketplace or other export restrictions.
- You need permission to create or reuse an AWS S3 bucket in the same AWS account used for export.
- You need a GCP project and a GCS bucket.
- You need `aws`, `gcloud`, and optionally `gsutil` installed locally.
- You need to identify which running EC2 instance in `us-west-1` should be turned into an AMI.

## Variables

Set these before running the commands:

```bash
export AWS_PROFILE=aws-4
export AWS_REGION=us-west-1
export INSTANCE_ID=i-00c5d187de48fd124
export AMI_NAME=migrated-from-ec2-$(date +%Y%m%d-%H%M%S)
export S3_PREFIX=exports/

export GCP_PROJECT=your-gcp-project
export GCS_BUCKET=your-gcs-bucket
export GCP_ZONE=us-west1-a
export IMAGE_NAME=imported-ami-image
```

Resolve the AWS account ID and derive the export bucket name from it:

```bash
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
  --profile "$AWS_PROFILE" \
  --query 'Account' \
  --output text)

export S3_BUCKET="joseph-aws-gcp-ami-$AWS_ACCOUNT_ID"
```

## Step 1: Scan `us-west-1` for Running EC2 Instances

List all running EC2 instances in AWS N. California:

```bash
aws ec2 describe-instances \
  --profile "$AWS_PROFILE" \
  --region "$AWS_REGION" \
  --filters Name=instance-state-name,Values=running \
  --query 'Reservations[].Instances[].{InstanceId:InstanceId,Name:Tags[?Key==`Name`]|[0].Value,State:State.Name,InstanceType:InstanceType,PrivateIp:PrivateIpAddress,PublicIp:PublicIpAddress,AZ:Placement.AvailabilityZone,ImageId:ImageId}' \
  --output table
```

If you already know the target instance, set it directly:

```bash
export INSTANCE_ID=i-00c5d187de48fd124
```

## Step 2: Create an AMI from the Running EC2 Instance

Create the AMI:

```bash
aws ec2 create-image \
  --profile "$AWS_PROFILE" \
  --region "$AWS_REGION" \
  --instance-id "$INSTANCE_ID" \
  --name "$AMI_NAME" \
  --no-reboot
```

The command returns a new `ImageId`. Save it:

```bash
export AMI_ID=ami-xxxxxxxxxxxxxxxxx
```

Wait for the AMI to become available:

```bash
aws ec2 wait image-available \
  --profile "$AWS_PROFILE" \
  --region "$AWS_REGION" \
  --image-ids "$AMI_ID"
```

## Step 3: Check or Create the S3 Export Bucket

```bash
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
  --profile "$AWS_PROFILE" \
  --query 'Account' \
  --output text)

export S3_BUCKET="joseph-aws-gcp-ami-$AWS_ACCOUNT_ID"

if aws s3api head-bucket \
  --profile "$AWS_PROFILE" \
  --bucket "$S3_BUCKET" 2>/dev/null; then
  echo "Using existing bucket: $S3_BUCKET"
else
  aws s3api create-bucket \
    --profile "$AWS_PROFILE" \
    --bucket "$S3_BUCKET" \
    --region "$AWS_REGION" \
    --create-bucket-configuration LocationConstraint="$AWS_REGION"
fi
```

This bucket naming rule is deterministic per AWS account:

```bash
joseph-aws-gcp-ami-{aws-account-id}
```

If the bucket already exists, keep using that same `S3_BUCKET` value for:

- `aws ec2 export-image` output into S3
- the later S3 to GCS copy step

## Step 4: Verify the AMI in AWS

```bash
aws ec2 describe-images \
  --profile "$AWS_PROFILE" \
  --region "$AWS_REGION" \
  --image-ids "$AMI_ID"
```

## Step 5: Create the AWS `vmimport` Role

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

## Step 6: Export the AMI to S3

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
s3://$S3_BUCKET/exports/export-ami-xxxx.vmdk
```

## Step 7: Prepare GCP

Authenticate and select the target project:

```bash
gcloud auth login
gcloud config set project "$GCP_PROJECT"
```

Create the destination GCS bucket:

```bash
gcloud storage buckets create "gs://$GCS_BUCKET" --location=us-west1
```

## Step 8: Copy the Exported Image from S3 to GCS

One simple approach is local relay:

```bash
mkdir -p ./exported-image

aws s3 cp "s3://$S3_BUCKET/$S3_PREFIX" ./exported-image/ \
  --recursive \
  --profile "$AWS_PROFILE"

gcloud storage cp ./exported-image/*.vmdk "gs://$GCS_BUCKET/"
```

If the image is large, consider a server-side transfer approach instead of pulling the file through your laptop.

## Step 9: Import the Disk Image into GCP

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

## Step 10: Launch a Test VM in GCP

```bash
gcloud compute instances create test-imported-vm \
  --zone="$GCP_ZONE" \
  --image="$IMAGE_NAME"
```

## Validation Checklist

- Confirm the export task completed successfully in AWS.
- Confirm the AMI was created from the intended running EC2 instance.
- Confirm the `.vmdk` file exists in GCS.
- Confirm the custom image exists in GCP.
- Boot a test VM from the image.
- Validate networking, disk mapping, startup behavior, and application health.

## Important Notes

- The starting point for this runbook is a running EC2 instance in `us-west-1`.
- AWS cannot export an AMI directly to GCS.
- The AWS-supported export target is S3 in the same AWS account.
- Not every AMI can be exported.
- Imported VMs often need guest OS adjustments after landing in GCP, especially around drivers, network configuration, and boot behavior.
- Windows imports may require additional driver and licensing review.
