# EC2 Instance Record

This repository stores the provisioning details for an EC2 instance created in AWS `us-west-1` (N. California).

## Instance Details

- Instance ID: `i-00c5d187de48fd124`
- AMI: `ami-00dabab179635c560`
- Instance type: `t3.medium`
- State: `running`
- Availability Zone: `us-west-1a`
- Public IP: `54.183.248.243`
- Public DNS: `ec2-54-183-248-243.us-west-1.compute.amazonaws.com`
- Private IP: `172.31.18.70`
- VPC: `vpc-0b66947b5f60ac18b`
- Subnet: `subnet-0889089b377e5889b`
- Security group: `sg-04c0b982769117112`
- Name tag: `codex-usw1-t3-medium-20260322`

## Notes

- Region: `us-west-1`
- The instance was launched in the default VPC and default security group.
- No key pair was attached at launch time.
- No inbound SSH rule was added during creation, so the instance is not directly reachable over SSH by default.
