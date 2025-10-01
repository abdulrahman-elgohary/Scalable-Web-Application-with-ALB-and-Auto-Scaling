
# Highly Available Web App on AWS (ALB + ASG + EC2)

Deploy a simple, scalable, and cost-optimized web application on AWS using **EC2**, **Application Load Balancer**, **Auto Scaling**, **Amazon RDS**, **IAM**, **CloudWatch**, and **SNS** all via the **AWS Console**
---

## ✅ Prerequisites

* AWS account with admin access to the Console
* A VPC with at least **two public subnets** (for ALB) and **two private subnets** (recommended for EC2/RDS). The default VPC works for quick tests.
* A **key pair** ([EC2] → Key Pairs → *Create key pair*) if you want SSH access.

---

## 1) Create an IAM Role for EC2

**Console path:** **IAM → Roles → Create role**

1. Trusted entity: **AWS service** → **EC2**.
2. Permissions:

   * **AmazonSSMManagedInstanceCore** (SSM Session Manager access—skip SSH keys)
   * **CloudWatchAgentServerPolicy** (if you’ll install CloudWatch Agent later)
   * (Optional) **AmazonS3ReadOnlyAccess** (if app reads S3), or a custom least-privilege policy.
3. Name: `EC2WebAppRole` → **Create role**.

> The role will be attached to instances via a Launch Template in the next steps.

---

## 2) Security Groups (SG)

Create **two** SGs.

### A) ALB Security Group

**Console path:** **EC2 → Security Groups → Create security group**

* Name: `ALBSG`
* Inbound:

  * `HTTP 80` from `0.0.0.0/0`
  * *(Optional)* `HTTPS 443` from `0.0.0.0/0` (requires ACM certificate later)
* Outbound: **All traffic** (default)

### B) EC2 (App) Security Group

* Name: `sg-ec2-app`
* Inbound:

  * `HTTP 80` **from** `sg-alb-web` (type: Custom, source: the ALB SG)
  * *(Optional for admin)* `SSH 22` from your IP or skip if using SSM
* Outbound: **All traffic** (default)

---

## 3) Target Group (for ALB health checks)

**Console path:** **EC2 → Target Groups → Create target group**

* Target type: **Instances**
* Protocol: **HTTP**, Port **80**
* VPC: your VPC
* Health checks: **HTTP**, path `/`
* Name: `Manara-TG`
* **Create** (don’t register targets yet; ASG will do it)

<img width="1627" height="720" alt="image" src="https://github.com/user-attachments/assets/97f93b55-fa6c-4867-833f-0f1317b89b74" />

---

## 4) Application Load Balancer (ALB)

**Console path:** **EC2 → Load Balancers → Create load balancer → Application Load Balancer**

* Name: `Manara-ALB`
* Scheme: **Internet-facing**
* IP type: **IPv4**
* Listeners: **HTTP:80**
* Availability Zones: pick **at least two** public subnets
* Security group: **`ALBSG`**
* Default action: **Forward** to **`Manara-TG`**
* **Create load balancer**

<img width="1640" height="707" alt="image" src="https://github.com/user-attachments/assets/8d564816-9a32-4dee-9bf8-84cccbf0a4ee" />

---

## 5) Launch Template (installs the web app)

**Console path:** **EC2 → Launch Templates → Create launch template**

* Name: `lt-webapp`
* AMI: **Amazon Linux 2** (or Amazon Linux 2023)
* Instance type: **t3.micro** (or **t3.small** for more headroom)
* Key pair: optional (if not using SSM)
* Network settings: don’t attach SG here (ASG step will); or attach **`sg-ec2-app`** if the UI prompts
* IAM instance profile: **`EC2WebAppRole`**
* User data (example “hello web app”):

```bash
#!/bin/bash
# Minimal web app install (Amazon Linux 2)
yum -y update
yum -y install httpd
cat >/var/www/html/index.html <<'HTML'
<!doctype html>
<html>
<head><title>Hello from ASG</title></head>
<body>
<h1>It works!</h1>
<p>Instance: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</p>
</body>
</html>
HTML
systemctl enable httpd
systemctl start httpd
```

* **Create launch template**

> Replace with your app’s install/start commands or container runtime as needed.

---

## 6) Auto Scaling Group

**Console path:** **EC2 → Auto Scaling Groups → Create Auto Scaling group**

1. **Choose launch template**: `lt-webapp`
2. **VPC & Subnets**: select **two or more** subnets (prefer **private** subnets if using a NAT Gateway; public subnets are fine for quick demo).
3. **Load balancing**:

   * Attach to an existing load balancer: **Yes**
   * Choose **Application Load Balancer**
   * Target group: **`tg-webapp`**
4. **Health checks**:

   * Turn on **ELB health checks** (in addition to EC2), grace period **300s**.
5. **Group size**:

   * Desired: **2**, Minimum: **2**, Maximum: **4** (example)
6. **Scaling policies** (simple rule of thumb):

   * **Scale out**: Add 1 instance if **Average CPU ≥ 60%** for 2 consecutive periods of 5 minutes
   * **Scale in**: Remove 1 instance if **Average CPU ≤ 30%** for 2 periods
     *(Choose “Target tracking” on **Average CPU 50–60%** for simpler ops.)*
7. **Instance distribution**: keep defaults; ensure **`sg-ec2-app`** is associated on the **Network** step if not set in the template.
8. **Create Auto Scaling group**.

After a few minutes, ASG will launch two instances, register them to the target group, and ALB should report **healthy**.

---

## 7) Amazon RDS (Multi-AZ)

**Console path:** **RDS → Databases → Create database**

* Engine: **MySQL**
* Templates: **Production**
* Multi-AZ: **Yes** (Multi-AZ DB instance)
* Credentials: Managed (Secret Manager)
* VPC & Subnets: choose **private subnets**
* Initiate a database and name it (Countries)
* Connectivity:

  * Public access: **No**
  * VPC security group: new SG `ExampleDB-SG` with inbound **TCP 3306** (MySQL) **from `Inventory-App`** (security-group-to-group reference)

<img width="1634" height="476" alt="image" src="https://github.com/user-attachments/assets/8d04b189-aa27-4c3d-a3c8-93edb8bf33fc" />

* **Create database**

<img width="1576" height="764" alt="image" src="https://github.com/user-attachments/assets/c0c966e4-ead0-4219-ae47-69847700907d" />


Update your app config (via user data, SSM Parameter Store, or Secrets Manager) to point to the RDS endpoint and port.

---

## 8) Monitoring, Alarms & Notifications

**SNS Topic**

* **SNS → Topics → Create topic** (type: Standard), name: `alerts-webapp`
* **Create subscription** → Protocol: **Email** → enter your email → **Confirm** the email you receive.

**CloudWatch Alarms**

* **CloudWatch → Alarms → All alarms → Create alarm**

  * **ALB 5XXErrorCount** > 0 for 5 minutes → notify `alerts-webapp`
  * **ASG Average CPU** ≥ 80% for 10 minutes → notify `alerts-webapp`
  * **TargetGroupUnHealthyHostCount** > 0 for 5 minutes → notify `alerts-webapp`

*(Optional)* Install **CloudWatch Agent** on instances to ship memory/disk metrics (use the `CloudWatchAgentServerPolicy` permission already attached).

---

## 9) HTTPS (Optional but recommended)

1. **ACM → Request a public certificate** for `app.example.com` (DNS validation with Route 53 is easiest).
2. **EC2 → Load Balancers → alb-webapp → Listeners → Add listener (HTTPS:443)**

   * Select the ACM certificate
   * Default action: **Forward** to `tg-webapp`
3. Add an **HTTP:80 → 301 Redirect to HTTPS** rule.

---

## 10) Test

* **EC2 → Load Balancers → alb-webapp → DNS name** (copy) → open in a browser.
* Refresh multiple times; you should see responses served by different instance IDs (from the user-data HTML).
* Trigger scale-out by running a CPU load test on instances (or temporarily lower the target tracking threshold), watch **ASG → Activity**.

---

## 🔒 Security & 💰 Cost Tips

* Prefer **private subnets** for EC2 instances; keep only the ALB public.
* Use **SSM Session Manager** instead of SSH and remove port 22.
* Store app secrets in **AWS Secrets Manager** or **SSM Parameter Store** (SecureString).
* Turn on **RDS automatic backups** and **deletion protection** (for prod).
* Enable **ALB access logs** and, if required, **WAF** for L7 protection.
* For demos, use **t3/t4g** burstables and set modest **ASG max** to cap cost.

---

## 🧹 Cleanup

1. **Auto Scaling Groups** → delete ASG
2. **Launch Templates** (if not used elsewhere) → delete
3. **Load Balancer** and **Target Group** → delete
4. **RDS** (if created) → take a final snapshot (optional) → delete
5. **Security Groups**, **IAM role** (if unused), **SNS topic**, **ACM cert**, **S3 log bucket** (empty then delete)

---

## 📁 Suggested Repo Structure

```
.
├── README.md                # This file
├── docs/
│   ├── architecture-diagram.png   # optional screenshot
│   └── screenshots/              # ALB, ASG, TG, alarms, etc.
├── user-data/
│   └── webapp-amzn-linux2.sh     # app install script (from above)
└── app/                           # your app code (if any)
```

---

## 🧩 Troubleshooting

* **ALB shows targets unhealthy**: confirm SG rules, health check path `/`, app listening on port 80, and instance User Data ran (`/var/log/cloud-init-output.log`).
* **ASG not attaching to TG**: verify ASG “Load balancing” step selected `tg-webapp`.
* **Cannot reach RDS**: check RDS SG inbound allows `sg-ec2-app` on the DB port; ensure instances can resolve RDS endpoint and have route/NAT for egress if in private subnets.

---

## License

MIT (adjust as needed)

---

> Tip: Commit this README to your GitHub repo and add screenshots under `docs/screenshots` that match each Console step to make the guide even easier to follow.
