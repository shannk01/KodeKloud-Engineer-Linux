# AWS 2-AZ VPC with Bastion Access and Auto Scaling Group
## Complete Step-by-Step Build Guide (with explanations and common-mistake corrections)

This guide builds: a VPC across 2 Availability Zones, public subnets with NAT Gateways and bastion JumpBoxes, private subnets running an Auto Scaling Group of Apache instances, fronted by an Application Load Balancer.

---

## Architecture at a glance

```
Internet
   |
[Internet Gateway]
   |
[ALB] -------------------------- public-subnet-A / public-subnet-B
   |                                   |        |
   v                              [JumpBox-A] [JumpBox-B]   (bastion hosts)
[Target Group]                    [NAT GW-A]  [NAT GW-B]
   |
   v
private-subnet-A / private-subnet-B
   |
[Auto Scaling Group] -> launches/replaces EC2 instances running Apache
```

You (your laptop) → JumpBox (public) → ASG instance (private). That's the only path into the private instances. The ALB is the only path the public can reach the app on.

---

## PART 1 — VPC

### What it is
A VPC is your own isolated network inside AWS — like your own private data center carved out of the cloud. Everything else (subnets, gateways, instances) lives inside it.

### Steps
1. Open **VPC console → Your VPCs → Create VPC**
2. Choose **VPC only** (not the wizard — we're building manually so you understand every piece)
3. Name: `my-vpc`
4. IPv4 CIDR block: `10.0.0.0/16` — this gives you 65,536 IP addresses to divide among subnets
5. Click **Create VPC**

### Correction — easy to miss
The "VPC only" option does **not** enable DNS hostnames automatically (the wizard does, this option doesn't).
- Select your VPC → **Actions → Edit VPC settings**
- Check **Enable DNS hostnames**
- Save

**Why this matters:** without DNS hostnames enabled, your instances won't get usable internal DNS names, which can break things like name resolution between instances later.

---

## PART 2 — Internet Gateway

### What it is
The IGW is the VPC's door to the public internet. Without it, nothing inside the VPC — not even public subnets — can reach or be reached from the internet.

### Steps
1. **VPC console → Internet Gateways → Create internet gateway**
2. Name: `my-igw`
3. Create it
4. Select it → **Actions → Attach to VPC** → choose `my-vpc`

### Correction
A very common mistake: creating the IGW but forgetting to attach it. If your public subnets can't reach the internet later, this is the first thing to check.

---

## PART 3 — Subnets

### What it is
Subnets are sub-divisions of your VPC's IP range, each tied to one Availability Zone. Public subnets will hold internet-facing things (ALB, NAT Gateways, JumpBoxes). Private subnets will hold your app instances, which should never be directly reachable from the internet.

### Steps
Go to **VPC console → Subnets → Create subnet**, select `my-vpc`, and create these one at a time:

| Name | Availability Zone | CIDR |
|---|---|---|
| public-subnet-A | AZ1 (e.g. us-east-1a) | 10.0.0.0/24 |
| public-subnet-B | AZ2 (e.g. us-east-1b) | 10.0.1.0/24 |
| private-subnet-A | AZ1 (same as public-A) | 10.0.2.0/24 |
| private-subnet-B | AZ2 (same as public-B) | 10.0.3.0/24 |

### Correction — important detail people get wrong
**private-subnet-A must be in the SAME AZ as public-subnet-A** (and likewise for B). This is because each private subnet routes its outbound traffic through the NAT Gateway *in its own AZ* — if they're mismatched, you either get cross-AZ data transfer charges or routing confusion.

After creating the two public subnets:
- Select each → **Actions → Edit subnet settings**
- Check **Enable auto-assign public IPv4 address**
- Save

Leave this **unchecked** for both private subnets — that's what makes them private.

---

## PART 4 — NAT Gateways

### What it is
A NAT Gateway lets instances in a private subnet (which have no public IP) reach the internet for things like software updates — while still preventing the internet from initiating connections *into* them. It's a one-way door, outbound only.

### Steps
Do this twice:

1. **VPC console → NAT Gateways → Create NAT Gateway**
2. Name: `NAT-public-subnetA`
3. Subnet: `public-subnet-A` (NAT Gateways live in public subnets, even though they serve private subnets)
4. Connectivity type: **Public**
5. Availability mode: **Zonal** (not Regional — Zonal gives you one NAT tied to one specific AZ, which matches a "one NAT per AZ" design; Regional auto-spans all AZs and is meant for simpler setups where you don't care about per-AZ control)
6. Click **Allocate Elastic IP** — this generates a new static public IP for the NAT to use
7. Create it
8. Repeat: `NAT-public-subnetB`, subnet `public-subnet-B`, its own new Elastic IP

### Correction — cost warning
NAT Gateways are **not free**: roughly $0.045/hour each, plus per-GB data processing charges. Two of them running 24/7 is about $65+/month combined. If this is a portfolio/practice build, delete the NAT Gateways (and release their Elastic IPs) when you're done testing, or AWS will keep billing you.

---

## PART 5 — Route Tables

### What it is
Route tables are the "directions" each subnet follows for outbound traffic. Without correct routes, subnets are isolated even if gateways exist.

### Steps
Create 3 route tables (**VPC console → Route Tables → Create route table**), each with VPC `my-vpc`:

**1. `rt-public`**
- Routes tab → Edit routes → Add route: Destination `0.0.0.0/0` → Target: `my-igw`
- Subnet associations tab → Edit → check `public-subnet-A` and `public-subnet-B`

**2. `rt-private-A`**
- Routes tab → Add route: Destination `0.0.0.0/0` → Target: `NAT-public-subnetA`
- Subnet associations → check `private-subnet-A` only

**3. `rt-private-B`**
- Routes tab → Add route: Destination `0.0.0.0/0` → Target: `NAT-public-subnetB`
- Subnet associations → check `private-subnet-B` only

### Correction — the #1 routing mistake
Using **one shared private route table** pointed at a single NAT Gateway for both private subnets. This works, but defeats the purpose of having 2 NAT Gateways (no AZ redundancy, plus possible cross-AZ charges). Always pair private-subnet-A with NAT-A, and private-subnet-B with NAT-B, in separate route tables.

---

## PART 6 — Security Groups

### What it is
Security groups are virtual firewalls attached to resources (not subnets). The key concept that trips people up: **a security group rule's "source" can be another security group, not just an IP address** — meaning "allow traffic from anything carrying that other security group," regardless of its actual IP.

### Steps
Create 3 security groups (**EC2 console → Security Groups → Create security group**), all in `my-vpc`:

**1. `sg-jumpbox`** (attach to both JumpBoxes)
- Inbound rule: Type SSH, Port 22, Source: **My IP** (auto-fills your current public IP)
- Outbound: leave default (all traffic)

**2. `sg-alb`** (attach to the ALB)
- Inbound rule: Type HTTP, Port 80, Source: **Anywhere-IPv4** (0.0.0.0/0) — the ALB must be reachable by anyone on the internet
- Outbound: default

**3. `sg-private`** (attach to the Launch Template used by the ASG)
- Inbound rule 1: Type SSH, Port 22, Source: start typing `sg-` in the source field and select **`sg-jumpbox`** from the dropdown
- Inbound rule 2: Type HTTP, Port 80, Source: select **`sg-alb`** from the dropdown
- Outbound: default

### Correction — what this actually achieves
With these rules, your private instances will **only** accept SSH from something that has `sg-jumpbox` attached (i.e., only the JumpBoxes — not your laptop directly, not the internet), and will **only** accept port 80 traffic from something that has `sg-alb` attached (i.e., only the load balancer — nobody can hit an instance's app port directly even with its private IP). This is what actually enforces "must go through the bastion" and "must go through the load balancer" — it's not just a convention, it's a hard technical restriction.

A common mistake: accidentally leaving SSH open to `0.0.0.0/0` on `sg-private` "just to test," then forgetting to remove it. Don't do this — the JumpBox-only rule should be the only SSH path, always.

---

## PART 7 — Key Pair

### Steps
1. **EC2 console → Key Pairs → Create key pair**
2. Name: `bastion-key`
3. Type: RSA, Format: `.pem`
4. Create — downloads automatically. **You cannot redownload this — save it safely.**
5. On your local machine: `chmod 400 bastion-key.pem`

### Correction
Don't choose "Proceed without a key pair." That option is meant for instances managed purely through AWS Systems Manager Session Manager (no SSH at all) — not for a bastion-based setup like this, where you need to actually SSH in.

---

## PART 8 — JumpBoxes (bastion hosts) — these stay manual, NOT in the ASG

### What it is
The ASG is only for your *application tier* (the instances behind the ALB). The JumpBoxes are a separate, fixed pair of small instances whose only job is to be your SSH entry point.

### Steps
Launch 2 EC2 instances manually (**EC2 console → Launch instance**):

**JumpBox-A**
- AMI: Amazon Linux 2023 or Ubuntu
- Instance type: `t2.micro`
- Key pair: `bastion-key`
- Network settings → Edit:
  - Subnet: `public-subnet-A`
  - Auto-assign public IP: **Enable**
  - Security group: select existing → `sg-jumpbox`
- User data: leave blank, or just `#!/bin/bash` + `sudo apt update -y && sudo apt upgrade -y`
- Launch

**JumpBox-B** — identical, but Subnet `public-subnet-B`. Reuse the **same** `sg-jumpbox` security group — don't create a second one (no benefit, just more to keep in sync).

### Correction
Don't install Apache or any app software on the JumpBoxes. They have one job — bastion access — and adding unnecessary software only increases their attack surface for no benefit.

---

## PART 9 — Target Group

### What it is
The Target Group is the list of instances the ALB sends traffic to, plus the health check rules that decide if an instance is "healthy" enough to receive traffic.

### Steps
1. **EC2 console → Target Groups → Create target group**
2. Target type: **Instances**
3. Name: `tg-private-instances`
4. Protocol: HTTP, Port: 80
5. VPC: `my-vpc`
6. Health checks:
   - Path: `/`
   - Healthy threshold: 2
   - Unhealthy threshold: 2
   - Timeout: 5 seconds
   - Interval: 30 seconds
7. Next → **don't register any targets manually** → Create target group

### Correction — important
Earlier guidance had you manually registering Instance-A/B here. **Don't do that with an ASG setup** — the ASG will register and deregister targets automatically as it launches/replaces instances. Manually adding targets here creates a setup the ASG doesn't fully control, which is a common source of confusing "healthy in console but not really" issues.

---

## PART 10 — Launch Template

### What it is
A blueprint the ASG uses every time it needs to launch a new instance — AMI, instance type, security group, key pair, and startup script, all bundled together.

### Steps
1. **EC2 console → Launch Templates → Create launch template**
2. Name: `lt-private-app`
3. AMI: Ubuntu
4. Instance type: `t2.micro`
5. Key pair: `bastion-key`
6. Network settings:
   - **Leave subnet unspecified/blank here** — the ASG controls which subnets to launch into, not the template
   - Security groups: select **`sg-private`**
7. Advanced details → User data:
```bash
#!/bin/bash
sudo apt update -y
sudo apt install -y apache2
echo "<h1>Server Details</h1><p><strong>Hostname:</strong> $(hostname)</p><p><strong>IP Address:</strong> $(hostname -I | cut -d' ' -f1)</p>" > /var/www/html/index.html
sudo systemctl restart apache2
```
8. Create launch template

### Correction
Setting a specific subnet inside the launch template is a common mistake — it conflicts with the ASG's own subnet settings and can cause instances to all land in one AZ instead of spreading across both. Leave it blank here; subnets get set at the ASG level in the next step.

---

## PART 11 — Auto Scaling Group

### What it is
The ASG launches and maintains a target number of instances based on the Launch Template, spread across the AZs you choose, and keeps them registered with the Target Group automatically — including replacing any that fail health checks.

### Steps
1. **EC2 console → Auto Scaling Groups → Create Auto Scaling group**
2. Name: `asg-private-app`
3. Launch template: `lt-private-app`
4. Network:
   - VPC: `my-vpc`
   - Subnets: select **both** `private-subnet-A` and `private-subnet-B`
5. Load balancing:
   - Choose **"Attach to an existing load balancer target group"**
   - Select `tg-private-instances`
6. Health checks:
   - Health check type: select **ELB** (not just the default EC2 check)
   - Health check grace period: **300 seconds**
7. Group size:
   - Desired capacity: 2
   - Minimum: 2
   - Maximum: 4 (or your choice)
8. Scaling policies: optional — skip for now, or add target tracking on CPU at 50% later
9. Create Auto Scaling group

### Correction — the exact issues you were likely hitting
- **Health check type left as "EC2" instead of "ELB":** with EC2-only checks, the ASG considers an instance healthy as long as it's simply *running* — even if Apache crashed or never started. Switching to **ELB** makes the ASG trust the actual ALB health check (is port 80 responding at `/`), which is what you want.
- **Grace period too short:** if left at the default (often very low), the ASG can mark a brand-new instance unhealthy and start replacing it before the user-data script has even finished installing Apache — creating an infinite replace loop. 300 seconds gives it breathing room.
- **Only one subnet selected:** if you only picked `private-subnet-A`, your ASG silently runs single-AZ even though you intended 2-AZ redundancy. Double-check both are checked.
- **Targets registered manually in Part 9, then also managed by the ASG:** causes duplicate/conflicting target registration. Keep target registration fully ASG-managed.

After creating it, check **EC2 → Instances** within a minute or two — you should see 2 new instances auto-launching, one per private subnet. Then check **Target Groups → tg-private-instances → Targets tab** — they should appear and turn **healthy** within the grace period.

---

## PART 12 — Application Load Balancer

### Steps
1. **EC2 console → Load Balancers → Create load balancer → Application Load Balancer**
2. Name: `my-alb`
3. Scheme: **Internet-facing**
4. VPC: `my-vpc`
5. Mappings: select **public-subnet-A** and **public-subnet-B**
6. Security group: remove the default, attach **`sg-alb`**
7. Listener: HTTP : 80 → forward to **`tg-private-instances`**
8. Create load balancer

---

## PART 13 — Test the App

1. Wait ~2–3 minutes for everything to settle
2. Copy the ALB's **DNS name** (EC2 → Load Balancers → my-alb)
3. Paste into a browser, refresh a few times — you should see the hostname/IP alternate between your two ASG instances
4. Try terminating one instance manually (**EC2 → Auto Scaling Groups → asg-private-app → Instance management** → terminate one) and watch the ASG automatically launch a replacement within a minute or two — this is the main payoff of using an ASG over manual instances

---

## PART 14 — Log in through the bastion

On your local machine:
```bash
ssh-add bastion-key.pem
ssh -A ec2-user@<JumpBox-public-ip>
```
Then, from inside the JumpBox:
```bash
ssh ec2-user@<asg-instance-private-ip>
```
Find current private IPs under **EC2 → Instances** — note these can change whenever the ASG replaces an instance, which is expected behavior.

**Optional — one-line login via SSH config**, on your local machine, edit `~/.ssh/config`:
```
Host jumpbox
    HostName <JumpBox-public-ip>
    User ec2-user
    IdentityFile ~/.ssh/bastion-key.pem

Host app-instance
    HostName <asg-instance-private-ip>
    User ec2-user
    ProxyJump jumpbox
    IdentityFile ~/.ssh/bastion-key.pem
```
Then just: `ssh app-instance`

---

## Summary checklist

- [ ] VPC created, DNS hostnames enabled
- [ ] IGW created and attached
- [ ] 4 subnets created, matched to correct AZs, public ones have auto-assign IP on
- [ ] 2 NAT Gateways (Zonal), each with its own Elastic IP, in the public subnets
- [ ] 3 route tables, each private one pointing to its own AZ's NAT
- [ ] 3 security groups, with `sg-private` referencing `sg-jumpbox` and `sg-alb` by SG (not IP)
- [ ] Key pair created and downloaded
- [ ] 2 JumpBoxes launched manually, both using `sg-jumpbox`
- [ ] Target group created with no manually-registered targets
- [ ] Launch template created with no subnet set, using `sg-private`
- [ ] ASG created across both private subnets, attached to target group, health check type = ELB, grace period 300s
- [ ] ALB created across both public subnets, using `sg-alb`, forwarding to target group
- [ ] Tested via ALB DNS name, confirmed instance replacement works
- [ ] Confirmed bastion-only SSH access works
