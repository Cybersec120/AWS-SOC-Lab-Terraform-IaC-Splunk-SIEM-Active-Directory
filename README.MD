🛡️ AWS SOC Lab — Terraform IaC | Splunk SIEM | Active Directory
A production-grade Security Operations Center lab deployed entirely via Terraform on AWS — featuring Splunk Enterprise, a Windows Active Directory domain, universal log forwarding, and red team attack simulations mapped to real-world threat scenarios.
<br>
    

📋 Table of Contents
Project Overview
Architecture
Tech Stack
Prerequisites
Deployment
Accessing Splunk
Attack Simulations
Dashboard Setup
Security Design
Project Structure
Cleanup

🔍 Project Overview
This lab replicates a real enterprise SOC environment — built from scratch using infrastructure-as-code — to develop and demonstrate skills in:
SIEM engineering — Splunk Enterprise with custom dashboards and search queries
Log aggregation — Universal Forwarders shipping Windows Event Logs from a live AD domain
Threat detection — Identifying brute force, privilege escalation, and lateral movement patterns
Cloud security — KMS encryption at rest, Secrets Manager credential management, SSM-only access (no exposed RDP/SSH)
Red team simulation — PowerShell attack scripts that generate realistic telemetry for blue team analysis


🧰 Tech Stack
Category
Technology
IaC
Terraform >= 1.5
Cloud
AWS EC2, VPC, KMS, Secrets Manager, S3, IAM, SSM, NAT Gateway
SIEM
Splunk Enterprise 9.2.1
Log Shipping
Splunk Universal Forwarder
Domain
Windows Server 2022 — Active Directory (AD DS)
Endpoint
Windows Server 2022 — Domain Workstation
OS (Splunk)
Ubuntu 22.04 LTS
Simulation
PowerShell attack scripts


✅ Prerequisites
# Required tooling
terraform  >= 1.5.0
aws-cli    >= 2.x         # Configured with appropriate IAM permissions

AWS IAM permissions required:
Service
Actions
EC2
Full (instances, VPC, SGs, key pairs, EIP)
KMS
CreateKey, Encrypt, Decrypt, GenerateDataKey
Secrets Manager
CreateSecret, GetSecretValue
S3
CreateBucket, PutObject, GetObject
SSM
StartSession, SendCommand
IAM
CreateRole, AttachRolePolicy, CreateInstanceProfile

Configure AWS CLI:
aws configure
# AWS Access Key ID: <your key>
# AWS Secret Access Key: <your secret>
# Default region: us-east-1
# Output format: json


🚀 Deployment
1. Clone the repository
git clone https://github.com/<your-username>/soc-lab.git
cd soc-lab

2. Review and configure variables
cp terraform.tfvars.example terraform.tfvars

Edit terraform.tfvars:
aws_region        = "us-east-1"
environment       = "soc-lab"
splunk_instance_type = "t3.large"
windows_instance_type = "t3.medium"
allowed_ip        = "<your-public-ip>/32"   # Restricts Splunk UI access

3. Initialize Terraform
terraform init

4. Review the execution plan
terraform plan -out=tfplan

5. Deploy the lab
terraform apply tfplan

⏱️ Estimated deployment time: 12–18 minutes
 AWS will provision EC2 instances, configure the AD domain, join WS01 to soc.lab, install Splunk, and deploy Universal Forwarders automatically via user data scripts.
6. Capture outputs
terraform output

Expected output:
splunk_public_ip     = "52.206.114.201"
splunk_url           = "http://52.206.114.201:8000"
dc01_instance_id     = "i-0abc123..."
ws01_instance_id     = "i-0def456..."
kms_key_arn          = "arn:aws:kms:us-east-1:..."


🔐 Accessing Splunk
Web UI
Navigate to the Splunk URL from Terraform outputs:
http://<splunk_public_ip>:8000

Retrieve credentials from AWS Secrets Manager:
# Splunk admin password
aws secretsmanager get-secret-value \
  --secret-id soc-lab/splunk/admin \
  --query SecretString \
  --output text

# Windows domain admin password
aws secretsmanager get-secret-value \
  --secret-id soc-lab/windows/admin \
  --query SecretString \
  --output text

SSH to Splunk (via SSM — no open port 22)
aws ssm start-session --target <splunk_instance_id>

RDP to Windows hosts (via SSM Port Forwarding)
# Forward local port 13389 to DC01 port 3389
aws ssm start-session \
  --target <dc01_instance_id> \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["3389"],"localPortNumber":["13389"]}'

# Connect with RDP client to: localhost:13389
# Domain: soc.lab  |  User: Administrator


⚔️ Attack Simulations
All simulations are located in attack-simulations/. They are designed to generate realistic Windows Event Log telemetry for blue team detection practice.
⚠️ Run these only within your lab environment. These scripts are for authorized simulation only.
Access a Windows host via SSM
aws ssm start-session --target <ws01_instance_id>

Then open PowerShell and run simulations:

Simulation 01 — Brute Force Attack
# Generates Event ID 4625 (Failed Logon) at high volume
.\attack-simulations\01_brute_force.ps1

What it generates:
Multiple failed authentication attempts against local and domain accounts
Source: WS01 → DC01
Splunk detection query:
index=windows EventCode=4625
| stats count by src_ip, user, _time
| where count > 10
| sort -count


Simulation 02 — Account Lockout
# Triggers Event ID 4740 (Account Locked Out)
.\attack-simulations\02_account_lockout.ps1

What it generates:
Account lockout events on the domain controller
Lockout source identification data
Splunk detection query:
index=windows EventCode=4740
| table _time, user, src, ComputerName
| sort -_time


Simulation 03 — Privilege Escalation
# Generates Event IDs 4672, 4728, 4732 (privilege assignment / group modification)
.\attack-simulations\03_privilege_escalation.ps1

What it generates:
User added to local Administrators group
Special privilege assignment at logon
Sensitive group membership changes
Splunk detection query:
index=windows (EventCode=4728 OR EventCode=4732 OR EventCode=4672)
| eval action=case(EventCode==4728,"Added to Global Group",EventCode==4732,"Added to Local Group",EventCode==4672,"Special Privileges Assigned")
| table _time, user, action, ComputerName


Simulation 04 — Lateral Movement
# Generates Event IDs 4648, 4624 Type 3 (network logon / pass-the-hash pattern)
.\attack-simulations\04_lateral_movement.ps1

What it generates:
Explicit credential usage across hosts
Network logon type 3 events
Remote service creation patterns
Splunk detection query:
index=windows EventCode=4624 Logon_Type=3
| stats count by src_ip, user, ComputerName
| where count > 3
| sort -count


Run All Simulations (Full Attack Chain)
# Execute the full kill chain — brute force → lockout → escalate → move laterally
$scripts = @(
    ".\attack-simulations\01_brute_force.ps1",
    ".\attack-simulations\02_account_lockout.ps1",
    ".\attack-simulations\03_privilege_escalation.ps1",
    ".\attack-simulations\04_lateral_movement.ps1"
)
foreach ($s in $scripts) { & $s; Start-Sleep -Seconds 5 }


📊 Dashboard Setup
The pre-built dashboard XML is located at soc_dashboard.xml.
Import via Splunk UI
Log in to Splunk at http://<splunk_ip>:8000
Navigate to: Search & Reporting → Dashboards → Create New Dashboard
Select "Classic Dashboard"
Click "Source" (top-right toggle)
Paste the contents of soc_dashboard.xml
Click Save
Import via Splunk REST API
SPLUNK_IP="52.206.114.201"
SPLUNK_PASS=$(aws secretsmanager get-secret-value \
  --secret-id soc-lab/splunk/admin \
  --query SecretString --output text)

curl -k -u admin:$SPLUNK_PASS \
  https://$SPLUNK_IP:8089/servicesNS/admin/search/data/ui/views \
  -d "name=soc_lab_dashboard" \
  --data-urlencode "eai:data@soc_dashboard.xml"

Dashboard Panels
Panel
Event IDs Monitored
Description
Failed Logons Over Time
4625
Brute force trend line
Top Locked-Out Accounts
4740
Account lockout frequency
Privilege Escalation Events
4728, 4732, 4672
Sensitive group changes
Lateral Movement Heatmap
4624 (Type 3), 4648
Cross-host auth patterns
Top Source IPs
4625, 4648
Attack origin ranking
Recent Security Events
All
Live event feed


🔒 Security Design
This lab implements multiple AWS security best practices:
Control
Implementation
Encryption at rest
KMS CMK applied to all EBS volumes and S3 buckets
Secrets management
All passwords stored in AWS Secrets Manager — never in Terraform state as plaintext
No exposed SSH/RDP
All access via AWS SSM Session Manager — ports 22 and 3389 are not open to the internet
Least privilege IAM
Separate instance profiles for Splunk and Windows hosts with scoped policies
Network isolation
Windows hosts reside in a private subnet with no direct internet exposure; outbound via NAT Gateway only
Security Groups
Splunk UI (8000) restricted to analyst IP; UF port (9997) restricted to internal VPC CIDR


📁 Project Structure
soc-lab/
├── providers.tf              # AWS provider + backend config
├── variables.tf              # Input variable declarations
├── terraform.tfvars          # Environment-specific values
├── main.tf                   # VPC, KMS, Secrets Manager, Key Pairs, Security Groups
├── splunk.tf                 # Splunk EC2, IAM role, S3 bucket, Elastic IP
├── windows.tf                # DC01, WS01, IAM roles, AD provisioning user data
├── outputs.tf                # Splunk URL, instance IDs, KMS ARN
├── soc_dashboard.xml         # Pre-built Splunk dashboard
└── attack-simulations/
    ├── 01_brute_force.ps1        # Failed logon flood (4625)
    ├── 02_account_lockout.ps1    # Account lockout trigger (4740)
    ├── 03_privilege_escalation.ps1 # Group membership abuse (4728/4732/4672)
    └── 04_lateral_movement.ps1   # Network logon + explicit creds (4624/4648)


🧹 Cleanup
Destroy all lab resources when finished to avoid ongoing charges:
terraform destroy

💡 Cost note: This lab uses t3.large (Splunk) and t3.medium (x2 Windows) instances. Running 24/7 in us-east-1 costs approximately $8–12/day. Destroy when not in use or use an EC2 Instance Scheduler.

🧠 Skills Demonstrated
Terraform IaC (multi-resource, multi-file, modular structure)
AWS infrastructure design (VPC, subnets, routing, NAT, IGW)
AWS security services (KMS, Secrets Manager, SSM, IAM least-privilege)
Splunk administration (indexer config, UF deployment, dashboard XML)
Active Directory deployment and domain join automation via user data
Windows event log analysis (Security, System, Application channels)
Red team TTP simulation (MITRE ATT&CK: T1110, T1078, T1021, T1134)

👤 Author
A.W. F. Shabazz El
 Cloud Architect & Security Consultant
 AWS Certified Solutions Architect · Azure Security Engineer · OSCP · CKA


Built with purpose. Documented with precision. Deployed with code.

