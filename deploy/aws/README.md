# Deploy CISO Assistant on AWS (EC2 + Docker Compose)

`ciso-assistant-ec2.yaml` is a CloudFormation template that runs the full stack
(PostgreSQL, backend, huey, frontend, Caddy) on one EC2 instance, with
automatic Let's Encrypt TLS. No AWS Amplify is involved.

What it creates:

- A dedicated VPC with one public subnet
- A security group that only allows 80 and 443 (no SSH; you get a shell through SSM Session Manager)
- A `t3.large` Ubuntu 24.04 instance with an encrypted gp3 volume and IMDSv2 required
- An Elastic IP, so the DNS record never has to change
- Daily EBS snapshots kept for 14 days (Data Lifecycle Manager)

Secrets (`POSTGRES_PASSWORD`, `DJANGO_SECRET_KEY`) are generated on first
boot and stored in `/opt/ciso-assistant/.env` (mode 600) on the instance.

## Prerequisites

- An AWS account and the AWS CLI v2, logged in (`aws sts get-caller-identity` works)
- A domain name you can add a DNS record for
- For shell access: the [Session Manager plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html) for the AWS CLI

## 1. Create the stack

```sh
aws cloudformation deploy \
  --stack-name ciso-assistant \
  --template-file deploy/aws/ciso-assistant-ec2.yaml \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides DomainName=grc.example.com
```

Optional parameters: `InstanceType` (default `t3.large`), `VolumeSize`
(default 40 GiB), `ImageTag` (default `latest`; pin a release for production),
and `AllowedCidr` to restrict who can reach HTTPS (e.g. your office VPN range).

## 2. Point DNS at the Elastic IP

```sh
aws cloudformation describe-stacks --stack-name ciso-assistant \
  --query "Stacks[0].Outputs" --output table
```

Create an `A` record for your domain with the `ElasticIP` value. If the domain
is in Route 53, do it in the hosted zone. Caddy keeps retrying until the record
resolves, then gets the certificate on its own.

## 3. Create the first admin user

The first boot takes about 5 minutes: installing Docker, pulling images, and
running migrations.

```sh
aws ssm start-session --target <InstanceId>
sudo -i
cd /opt/ciso-assistant
docker compose ps                     # wait until backend shows "healthy"
docker compose exec backend python manage.py createsuperuser
```

Then open `https://<your domain>`.

## Operations

All commands run from `/opt/ciso-assistant` in an SSM session.

| Task | Command |
| --- | --- |
| Logs | `docker compose logs -f backend` |
| Upgrade | `docker compose pull && docker compose up -d` (or change `TAG` in `.env` first) |
| Database dump | `docker compose exec postgres pg_dump -U ciso_assistant ciso_assistant > backup.sql` |
| Boot script log | `cat /var/log/cloud-init-output.log` |

The root volume uses `DeleteOnTermination: false`, so deleting the stack keeps
the data volume. Delete it by hand once you no longer need it.

## Going further

- **Email**: add `EMAIL_HOST`, `EMAIL_PORT`, `EMAIL_HOST_USER`,
  `EMAIL_HOST_PASSWORD`, `EMAIL_USE_TLS` and `DEFAULT_FROM_EMAIL` to the
  `x-backend-env` block in `docker-compose.yml` (Amazon SES SMTP works). See
  `product-docs/installation/mailer.md`.
- **Managed database**: create an RDS PostgreSQL instance, set `DB_HOST`,
  `POSTGRES_*` and `POSTGRES_SSL_MODE=require` in the backend env, and drop the
  `postgres` service.
- **Attachments on S3**: set `USE_S3=True`, `AWS_STORAGE_BUCKET_NAME` and
  `AWS_S3_REGION_NAME`, then give the instance role access to the bucket. See
  `product-docs/installation/s3.md`.
- **Kubernetes**: on EKS, use the Helm chart in `charts/ciso-assistant-next`
  instead of this template.
