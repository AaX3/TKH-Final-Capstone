# Secure Automated Web Architecture

## Description
This project provisions a secure, automated web server infrastructure on AWS using Terraform. It builds a custom VPC with public networking, deploys a hardened EC2 instance running Apache via an automated bootstrap script, and enforces security compliance through a CI/CD pipeline that statically scans all infrastructure code for misconfigurations before deployment.

## Technologies Used
- AWS (VPC, EC2, EBS, Security Groups, IAM)
- Terraform
- GitHub Actions
- tfsec

## Architecture
The network is built on a custom VPC (`10.0.0.0/16`) with DNS hostnames enabled and a single public subnet (`10.0.1.0/24`) that auto-assigns public IPs to launched instances. An Internet Gateway and associated route table (`0.0.0.0/0` → IGW) provide outbound internet connectivity for the subnet.

Access is controlled through a dedicated security group (`tkh-web-security-group`) with explicit, individually-defined rules rather than broad defaults:
- **Inbound HTTP (port 80)** is open to `0.0.0.0/0`, since the web server is intended to be publicly reachable.
- **Inbound SSH (port 22)** is restricted to a single trusted home IP address (`var.my_home_ip`, `/32`), supplied via a gitignored `terraform.tfvars` file rather than hardcoded — keeping the actual IP out of version control while still enforcing the restriction in code.
- **Outbound traffic** is unrestricted, allowing the instance to reach AWS endpoints and package repositories during bootstrap.

The EC2 instance (`t2.micro`) includes several additional security measures beyond the network setup. It automatically uses the latest official Amazon Linux 2023 image instead of a fixed, outdated one (via a Terraform data source), so the server always launches with up-to-date software. It also requires a more secure method for the instance to access its own AWS metadata (IMDSv2, enforced with `http_tokens = "required"`), which closes off a common attack path used to steal cloud credentials. The instance's storage drive is also encrypted at rest (`encrypted = true`), so the data on it is protected even if the underlying storage were ever compromised.

Setup is fully automated: when the instance first boots, a `user_data` startup script installs and starts the Apache web server (`httpd`) on its own — no manual login or configuration is needed after running `terraform apply`.

Before any changes can be deployed, a GitHub Actions pipeline (`security-scan.yml`) automatically scans the Terraform code with `tfsec` every time it's pushed to the `main` branch. With `soft_fail: false`, if the scanner finds a problem, the pipeline fails outright rather than just warning about it — so insecure code can't move forward until it's actually fixed.

Link to Presentation: https://drive.google.com/drive/u/0/folders/1I9uonlMbsF9lXESaC4Ke-_RpB3XsH4g- 