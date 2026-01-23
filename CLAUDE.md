# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an AWS Lambda function that runs Certbot to obtain and renew SSL/TLS certificates from Let's Encrypt. It supports wildcard certificates using DNS-01 challenges and uploads the resulting certificates to S3. The Lambda is triggered by CloudWatch scheduled events for automated certificate renewal.

## Key Architecture

### Lambda Handler Flow (main.py)
The Lambda handler follows this sequence:
1. `lambda_handler()` - Entry point that wraps execution with cleanup
2. `guarded_handler()` - Main logic that:
   - Reads DNS plugin from `DNS_PLUGIN` environment variable
   - Reads certificate parameters from CloudWatch event payload (emails, domains, S3 bucket/prefix/region)
   - Calls `obtain_certs()` to run Certbot with DNS-01 challenge
   - Calls `upload_certs()` to upload resulting certificates to S3
3. `rm_tmp_dir()` - Cleanup called before and after execution to clear `/tmp/certbot`

### Certbot Integration
- Uses `/tmp/certbot` as working directory (Lambda's writable temp space)
- Runs in non-interactive mode with DNS-01 challenge
- Certificate structure in `/tmp/certbot/live/[domain]/`:
  - cert.pem, chain.pem, fullchain.pem, privkey.pem, README

### Terraform Infrastructure (terraform/)
- Creates Lambda function with IAM role
- Provisions Route53 permissions for DNS challenge (if `create_aws_route53_iam_role=true`)
- Provisions S3 upload permissions (if `create_aws_s3_iam_role=true`)
- Creates CloudWatch Event Rules for each certificate in `certificates` variable
- Each event rule triggers Lambda with specific certificate parameters

The `certificates` variable is a map where each entry defines:
- Scheduling (cron/rate expression)
- Certificate parameters (emails, domains, S3 destination)

## Development Commands

### Build Lambda Package
```bash
./package.sh
```
This creates `certbot/certbot-lambda.zip` containing:
1. Python 3.13 virtualenv with all requirements.txt dependencies
2. main.py Lambda handler
Uses Amazon Linux 2023 Python 3.13 (matching Lambda runtime)

### Update Certbot Version
1. Create/activate virtualenv:
```bash
python3.13 -m venv certbot/venv
source certbot/venv/bin/activate
```

2. Install desired versions:
```bash
pip3 install certbot==<VERSION> certbot-dns-route53==<VERSION> [other-plugins]
```

3. Generate new requirements.txt:
```bash
pip freeze | grep -v "pkg-resources" > requirements.txt
```

**Note:** Ensure dependency versions are compatible with Python 3.13 (e.g., cffi>=2.0.0)

### Deploy with Terraform
```bash
cd terraform
terraform init
terraform plan
terraform apply
```

## Environment Variables

### Lambda Environment
- `DNS_PLUGIN` - Certbot DNS plugin name (default: "dns-route53")
- `CERTBOT_URL` - Let's Encrypt ACME endpoint (default: production v02 API)
- `KEY_TYPE` - Certificate key algorithm type. Valid values: 'rsa' or 'ecdsa' (recommended default: "ecdsa"). ECDSA keys are smaller and more efficient than RSA keys.
- Custom variables (e.g., for non-AWS DNS providers like Tencent Cloud)

### Event Payload (from CloudWatch Event)
- `emails` - Comma-separated email addresses for Let's Encrypt registration
- `domains` - Comma-separated domain list (supports wildcards like *.example.com)
- `s3_bucket` - Destination S3 bucket name
- `s3_prefix` - Key prefix for uploaded certificates
- `s3_region` - AWS region of S3 bucket

## DNS Plugin Support

The codebase supports multiple DNS providers through Certbot plugins:
- **Route53** (default): Built-in AWS support, uses IAM role permissions
- **Other providers**: Specify plugin in `certbot_dns_plugin` variable and provide credentials via `lambda_custom_environment`

Example for Tencent Cloud:
```hcl
certbot_dns_plugin = "dns-tencentcloud"
lambda_custom_environment = {
  TENCENTCLOUD_SECRET_ID  = "..."
  TENCENTCLOUD_SECRET_KEY = "..."
}
```

## CI/CD

GitHub Actions workflow (`.github/workflows/release.yml`):
- Triggers on version tags (v*)
- Builds in Amazon Linux 2023 container
- Runs `package.sh` to create Lambda zip
- Creates GitHub release with `certbot-lambda.zip` artifact

## Important Implementation Notes

- Lambda runtime uses Python 3.13 on Amazon Linux 2023
- Build environment must match: use `amazonlinux:latest` (AL2023) or `public.ecr.aws/lambda/python:3.13`
- The Lambda requires sufficient timeout (default 300s) for DNS propagation and ACME challenge
- IAM permissions needed: Route53 (ListHostedZones, GetChange, ChangeResourceRecordSets) and S3 (ListBucket, PutObject)
- Certificates are organized by domain name in S3: `{prefix}/{domain}/cert.pem`, etc.
- ACME account state is tied to email address - using the same email across different Lambda functions may cause "No such authorization" errors if previous runs left stale state
