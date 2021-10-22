# certbot-lambda

Running Certbot on AWS Lambda.

Inspired by [Deploying EFF's Certbot in AWS Lambda](https://arkadiyt.com/2018/01/26/deploying-effs-certbot-in-aws-lambda/).

## Features

- Supports wildcard certificates (Let's Encrypt ACME v2).
- Uploads certificates to specified Amazon S3 bucket.
- Works with CloudWatch Scheduled Events for certificate renewal.
- Use Terraform to deploy to AWS (See [terraform folder](terraform)).

## How to archive zip file for lambda function
```bash
./package.sh
```

## Variables

### Certificates
This is a map of certificates that you want to schedule renewals for (using cloudwatch events)

| Variable            | Description                                                                                                            | Type   | Required |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------- | ------ | -------- |
| name                | name for the cloudwatch event                                                                                          | string | y        |
| description         | description for the cloudwatch event                                                                                   | string | y        |
| schedule_expression | schedule in [cron() or rate() format](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/ScheduledEvents.html) | string | y        |
| is_enabled          | enable/disable switch                                                                                                  | bool   | y        |
| input               | the parameters that get sent to the lambda                                                                             | map    | y        |


### Input
| Variable  | Description                                                 | Type   | Required |
| --------- | ----------------------------------------------------------- | ------ | -------- |
| emails    | comma-separated list of email to associate with the LE cert | string | y        |
| domains   | comma-separated list of domains to request certs for        | string | y        |
| s3_bucket | s3 bucket to save the certificate files in                  | string | y        |
| s3_prefix | s3 prefix to save the certifcate files with                 | string | y        |
| s3_region | region of the s3 bucket                                     | string | y        |

### Example
```
certificates = {
  "my_cert" = {
    name : "my_cert"
    description : "SSL certificate for my site"
    schedule_expression : "cron(0 16 L * ? *)"
    is_enabled : true
    input = {
      emails : "admin@mysite.com"
      domains : "www.mysite.com"
      s3_bucket : "my_s3_bucket"
      s3_prefix : "certs/"
      s3_region : "us-east-1"
    }
  }
}
```

## How to update certbot version

- Source virtualenv
```bash
source certbot/venv/bin/activate
```
- Recreate requirements.txt with any plugins
```bash
readonly CERTBOT_VERSION=1.17.0
readonly CERTBOT_DNS_TENCENTCLOUD_VERSION=1.3.0
pip3 install \
    certbot==${CERTBOT_VERSION} \
    certbot-dns-route53==${CERTBOT_VERSION} \ 
    certbot-dns-tencentcloud==${CERTBOT_DNS_TENCENTCLOUD_VERSION} # Optional dns plugin
```
- Create new requirements file
```bash
# https://stackoverflow.com/questions/39577984/what-is-pkg-resources-0-0-0-in-output-of-pip-freeze-command
pip freeze | grep -v "pkg-resources" > requirements.txt
```