# Auto Scaling + Application Load Balancer on AWS

A self-healing, highly available web infrastructure on AWS free tier using EC2 Auto Scaling and an Application Load Balancer — no manual server management required.

![AWS](https://img.shields.io/badge/AWS-Auto%20Scaling%20%2B%20ALB-orange?logo=amazon-aws) ![Free Tier](https://img.shields.io/badge/Free%20Tier-Eligible-green) ![High Availability](https://img.shields.io/badge/High%20Availability-Multi--AZ-blue)

---

## Architecture

```
Internet
    │
    ▼
Application Load Balancer (internet-facing, port 80)
    │                    │
    ▼                    ▼
EC2 t2.micro         EC2 t2.micro
Amazon Linux 2       Amazon Linux 2
Nginx + index.html   Nginx + index.html
AZ: ap-south-1a      AZ: ap-south-1b
    │                    │
    └────────┬───────────┘
             ▼
        S3 Bucket
     (source of truth)
     stores index.html
```

Every EC2 instance pulls `index.html` from S3 on boot via the user data script. If an instance goes down, Auto Scaling automatically launches a replacement — no manual action needed.

---

## Tech Stack

| Service | Purpose |
|---|---|
| EC2 t2.micro | Runs Nginx web server |
| Auto Scaling Group | Maintains desired instances, replaces failed ones |
| Application Load Balancer | Distributes traffic across healthy instances |
| Launch Template | Blueprint for every EC2 instance ASG creates |
| Target Group | Pool of instances ALB routes traffic to |
| S3 | Stores website files — single source of truth |
| IAM Role | Allows EC2 to read S3 without hardcoded credentials |
| Custom VPC | Isolated network with two public subnets |
| Security Groups | ALB accepts internet traffic, EC2 only accepts ALB traffic |
| User Data script | Runs on boot — installs Nginx, copies files from S3 |
| Instance Metadata Service (IMDSv2) | Fetches instance ID and AZ dynamically at runtime |

---

## Project Structure

```
project5/
├── user-data.sh         # EC2 startup script (runs on every new instance)
├── index.html           # Website file (uploaded to S3)
└── README.md
```

---

## Key Concepts Learned

- **Launch Template** — a reusable blueprint containing AMI, instance type, security group, IAM role, and user data script
- **Auto Scaling Group** — maintains a desired number of healthy instances; automatically replaces terminated or unhealthy ones
- **User Data script** — bash script that runs once on every new EC2 instance at first boot
- **Application Load Balancer** — routes HTTP traffic across all healthy instances; internet-facing vs internal
- **Target Group + health checks** — ALB pings `/index.html` every 30 seconds; unhealthy instances are removed from rotation
- **Multi-AZ deployment** — instances spread across two AZs so one AZ failure doesn't take down the site
- **IMDSv2** — secure way to fetch instance metadata; requires a token unlike the older IMDSv1
- **S3 as source of truth** — every instance pulls files from S3 on boot; updating the site means updating S3 and refreshing instances
- **Security group layering** — `alb-sg` accepts internet traffic on port 80; `ec2-sg` only accepts traffic from `alb-sg`, not the public internet directly
- **Service-linked roles** — AWS automatically creates IAM roles like `AWSServiceRoleForAutoScaling`; these are managed by AWS and should not be deleted

---

## Infrastructure Setup

### Prerequisites
- AWS account (free tier)
- AWS CloudShell

---

### Step 1 — Upload website to S3

```bash
BUCKET="mysite-asg-yourname"
aws s3 mb s3://$BUCKET --region ap-south-1
aws s3 cp ~/index.html s3://$BUCKET/index.html
aws s3 ls s3://$BUCKET/
```

---

### Step 2 — Create custom VPC

| Resource | Value |
|---|---|
| VPC CIDR | 10.0.0.0/16 |
| Subnet 1 | 10.0.1.0/24 — ap-south-1a |
| Subnet 2 | 10.0.2.0/24 — ap-south-1b |
| Internet Gateway | Attached to VPC |
| Route table | 0.0.0.0/0 → IGW, associated with both subnets |

Enable auto-assign public IP on both subnets.

---

### Step 3 — Create security groups

**ALB security group (`alb-sg`):**
| Type | Port | Source |
|---|---|---|
| HTTP | 80 | 0.0.0.0/0 |

**EC2 security group (`ec2-sg`):**
| Type | Port | Source |
|---|---|---|
| HTTP | 80 | alb-sg |

EC2 instances are not directly reachable from the internet — only through the ALB.

---

### Step 4 — Create IAM role

- Name: `EC2S3ReadRole`
- Trusted entity: EC2
- Permission: `AmazonS3ReadOnlyAccess`

Attach to Launch Template so instances can pull files from S3 without hardcoded credentials.

---

### Step 5 — Create Launch Template

| Setting | Value |
|---|---|
| AMI | Amazon Linux 2 (free tier eligible) |
| Instance type | t2.micro |
| Security group | ec2-sg |
| IAM instance profile | EC2S3ReadRole |

User data script:

```bash
#!/bin/bash
yum update -y
yum install -y nginx awscli
systemctl start nginx
systemctl enable nginx

# IMDSv2 — fetch token first
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# Fetch instance metadata
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/availability-zone)

# Pull website file from S3
aws s3 cp s3://YOUR-BUCKET-NAME/index.html /usr/share/nginx/html/index.html

# Inject instance info into page
sed -i "s/INSTANCE_PLACEHOLDER/$INSTANCE_ID/g" /usr/share/nginx/html/index.html
sed -i "s/AZ_PLACEHOLDER/$AZ/g" /usr/share/nginx/html/index.html

chmod 644 /usr/share/nginx/html/index.html
```

---

### Step 6 — Create Target Group

| Setting | Value |
|---|---|
| Target type | Instances |
| Protocol | HTTP port 80 |
| VPC | asg-vpc |
| Health check path | /index.html |
| Healthy threshold | 2 checks |
| Interval | 30 seconds |

---

### Step 7 — Create Application Load Balancer

| Setting | Value |
|---|---|
| Scheme | **Internet-facing** |
| VPC | asg-vpc |
| Subnets | Both public subnets |
| Security group | alb-sg |
| Listener | HTTP 80 → forward to target group |

> Note: Always verify the scheme is **internet-facing** — an internal ALB is only reachable from inside the VPC and won't work from a browser. Internal ALBs have `internal-` prefix in their DNS name.

---

### Step 8 — Create Auto Scaling Group

| Setting | Value |
|---|---|
| Launch template | web-server-template |
| VPC | asg-vpc |
| Subnets | Both public subnets |
| Load balancer | Attach to target group |
| Health check | ELB health checks enabled |
| Desired capacity | 1 |
| Minimum | 1 |
| Maximum | 2 |

---

### Step 9 — Visit your site

```
http://YOUR-ALB-DNS-NAME.ap-south-1.elb.amazonaws.com
```

---

## Testing Auto Scaling (self-healing)

1. Go to **EC2 → Instances → terminate your running instance**
2. Watch **Auto Scaling Groups → Activity tab** — a new launch starts within 1–2 minutes
3. Watch **Target Groups → Targets** — new instance becomes healthy
4. Visit the ALB URL — site comes back automatically

This proves the infrastructure is self-healing with no manual intervention.

---

## Testing Load Balancing

1. Set **Auto Scaling Group desired capacity to 2**
2. Wait for both instances to show healthy in the target group
3. Keep refreshing the ALB URL
4. Instance ID and AZ change with each request — proving the ALB is distributing traffic

Set desired capacity back to 1 when done to stay within free tier.

---

## Updating the site

```bash
# Edit your file and push to S3
aws s3 cp ~/index.html s3://YOUR-BUCKET-NAME/index.html

# Trigger instance refresh in console:
# EC2 → Auto Scaling Groups → web-asg → Instance refresh → Start
```

New instances pull the updated file from S3 on boot.

---

## Free Tier Considerations

| Resource | Free tier limit | Notes |
|---|---|---|
| EC2 t2.micro | 750 hrs/month | Keep desired=1 to stay within limit |
| ALB | 750 hrs/month | 1 ALB is fine |
| S3 | 5 GB / 20k requests | Well within limits |
| VPC/IGW/SG | Free always | No cost |

> **Warning:** Setting desired capacity to 2 uses 1500 hrs/month — exceeds the 750 hr free tier limit. Only scale to 2 temporarily for testing, then set back to 1.

---

## Cleanup

Delete resources in this order to avoid dependency errors:

1. **Auto Scaling Group** → deletes EC2 instances automatically
2. **Load Balancer** → delete ALB
3. **Target Group** → delete
4. **Launch Template** → delete
5. **IAM Role** → delete `EC2S3ReadRole` only — leave AWS service-linked roles
6. **Security Groups** → delete `ec2-sg` first, then `alb-sg`
7. **S3** → empty bucket then delete
8. **VPC** → detach and delete IGW → delete subnets → delete VPC

---

## Common Mistakes and Fixes

| Mistake | Symptom | Fix |
|---|---|---|
| ALB set to internal | URL not reachable from browser | Delete and recreate as internet-facing |
| Wrong instance type in Launch Template | Non-free-tier instance launches | Edit template, change to t2.micro |
| No security group in Launch Template | ALB can't reach EC2 | Edit template, add ec2-sg |
| No IAM role attached | aws s3 cp fails silently | Edit template, attach EC2S3ReadRole |
| IMDSv1 blocked | curl metadata returns empty | Use IMDSv2 with token header |

---

## What's Next

| Project | What you'll learn |
|---|---|
| Project 6 — Lambda + API Gateway | Serverless backend, no EC2 needed |
| Project 7 — CI/CD Pipeline | Auto deploy on every git push |
| CloudFormation for this setup | Rebuild entire infrastructure as code |
