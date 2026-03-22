# AMI Migration n8n Orchestration

This document describes the current orchestration flow implemented in n8n and on the VM worker.

## Main Flow

```text
Manual Trigger
AMI Migration - Step 1 Dispatch Scan via Gateway
        |
        v
VM gateway -> Pub/Sub -> VM worker
        |
        v
AMI Migration - Step 1 Callback Receiver
select the first returned instanceId by default
        |
        v
auto enqueue create-ami job
        |
        v
VM gateway -> Pub/Sub -> VM worker
run create-image + wait image-available
        |
        v
AMI Migration - Step 2 Callback Receiver
AMI becomes available, then auto enqueue export-ami job
        |
        v
VM gateway -> Pub/Sub -> VM worker
run export-image
        |
        v
AMI Migration - Step 5 Callback Receiver
receive final export result when AWS completes the export
```

## Short Version

```text
Step 1 Dispatch -> Step 1 Callback -> auto Step 2 -> Step 2 Callback -> auto Step 5 -> Step 5 Callback
```

## Manual vs Automatic

- `AMI Migration - Step 1 Dispatch Scan via Gateway`
  This is the main entrypoint. You trigger this manually.
- `AMI Migration - Step 2 Dispatch Create AMI`
  This exists as a manual fallback, but the normal path auto-triggers it from Step 1 callback.
- `AMI Migration - Step 5 Dispatch Export AMI`
  This exists as a manual fallback, but the normal path auto-triggers it from Step 2 callback.

## Current Defaults

- AWS profile: `aws-4`
- AWS region: `us-west-1`
- Step 1 selection rule: choose the first returned running instance
- Export bucket: `ec2-ami-export-379810014062-20260322-usw1`
- Export prefix: `exports/`
- Export role: `vmimport`
- Export disk format: `VMDK`

## Worker Components

- Gateway service:
  `/home/joseph_siyi/ami-migration-worker/pubsub_gateway.py`
- Worker service:
  `/home/joseph_siyi/ami-migration-worker/pubsub_scan_worker.py`

These are managed by user-level `systemd` services:

- `ami-migration-gateway.service`
- `ami-migration-worker.service`

## n8n Workflows

- Step 1 Dispatch:
  `AMI Migration - Step 1 Dispatch Scan via Gateway`
- Step 1 Callback:
  `AMI Migration - Step 1 Callback Receiver`
- Step 2 Dispatch:
  `AMI Migration - Step 2 Dispatch Create AMI`
- Step 2 Callback:
  `AMI Migration - Step 2 Callback Receiver`
- Step 5 Dispatch:
  `AMI Migration - Step 5 Dispatch Export AMI`
- Step 5 Callback:
  `AMI Migration - Step 5 Callback Receiver`

## Notes

- Step 4 is no longer part of the workflow chain.
- The `vmimport` role and export bucket were created manually in AWS and are now reused by Step 5.
- During export, AWS may create `vmimportexport_write_verification.txt` in the bucket before the final disk image appears.
