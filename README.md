# AWS Hands-On Lab — Application Load Balancer (ALB) with EC2 & Target Groups

A hands-on AWS lab that demonstrates how to distribute HTTP traffic across two EC2 instances placed in **two different Availability Zones**, using an **Application Load Balancer (ALB)** and a **Target Group**, inside the **Default VPC**.

---

## 📌 Overview

This lab walks through building a highly available web tier on AWS without writing a single line of application code — everything is done through the AWS Console (EC2, VPC, Target Groups, ELB).

**Region used:** `us-east-1`
**VPC used:** Default VPC (`vpc-0ca9fcfcb323e78de`)
**Availability Zones used:** `us-east-1c` and `us-east-1d`

### What we build

1. Two EC2 instances (`server1`, `server2`) in **public subnets**, each in a different AZ.
2. Each instance runs **Apache (httpd)** via a **User Data script** that serves a simple identifying web page.
3. A **Target Group** that registers both instances as backend targets.
4. An **Application Load Balancer (ALB)** that sits in front of the Target Group and load-balances incoming traffic (Round Robin) between the two instances.

---

## 🗺️ Architecture Diagram

```
                                Internet
                                   │
                                   ▼
                         ┌───────────────────┐
                         │      Route 53      │
                         │ (DNS resolution for │
                         │   the ALB domain)   │
                         └─────────┬──────────┘
                                   │ resolves to
                                   ▼
                    ┌──────────────────────────────┐
                    │   Application Load Balancer   │
                    │           (ALB-1)             │
                    │   Listener: HTTP : 80          │
                    └───────────┬───────────┬───────┘
                                │           │
                    Node in AZ 1d           Node in AZ 1c
                                │           │
                                ▼           ▼
                 ┌───────────────────┐   ┌───────────────────┐
                 │   Target Group     │   │   Target Group     │
                 │      (TG-Test)     │   │      (TG-Test)     │
                 └─────────┬─────────┘   └─────────┬─────────┘
                           │                        │
                           ▼                        ▼
                 ┌───────────────────┐   ┌───────────────────┐
                 │   EC2 - server1    │   │   EC2 - server2    │
                 │  Public Subnet     │   │  Public Subnet     │
                 │  AZ: us-east-1d    │   │  AZ: us-east-1c    │
                 │  Apache (httpd)    │   │  Apache (httpd)    │
                 │  Default VPC       │   │  Default VPC       │
                 └───────────────────┘   └───────────────────┘
```

**Default VPC** spans across every Availability Zone in the region — that's why we can place `server1` in AZ `us-east-1d` and `server2` in AZ `us-east-1c`, both still inside the *same* VPC (`172.31.0.0/16`), just in different subnets/AZs.

---

## ⚙️ Prerequisites

- An AWS account with access to the EC2 and VPC console.
- Default VPC available in the region (`us-east-1` in this lab).
- Two public subnets (one per AZ) with a route to an Internet Gateway.
- A security group allowing inbound **HTTP (port 80)** from `0.0.0.0/0`.

---

## 🚀 Step-by-Step Walkthrough

### 1. Confirm subnets exist in the target AZs

Before launching anything, we confirm the Default VPC already has public subnets in `us-east-1c` and `us-east-1d`.

![Subnets list](images/Screenshot%202026-07-26%20052644.png)
*Two public subnets already exist in the Default VPC: `pub-subnet-az-1d` and `pub-subnet-az-1c`. Both are `Available` and belong to the same VPC (`vpc-0ca9fcfcb323e78de`).*

---

### 2. Launch EC2 Instance — `server1`

- Instance type: `t2.micro`
- **VPC:** Default VPC (`vpc-0ca9fcfcb323e78de`)
- **Subnet:** `pub-subnet-az-1c` / AZ `us-east-1c` (or `1d` for server1 — see network settings below)
- **Auto-assign Public IP:** Enable ✅ (mandatory, otherwise the instance is unreachable from the internet)
- **Security Group:** allow inbound HTTP (port 80)
- **User Data:** the Apache install + HTML page script (see [User Data Scripts](#-user-data-scripts) below)

![EC2 Network Settings](images/Screenshot%202026-07-26%20053222.png)
*Network settings screen while launching the instance: choosing the Default VPC, a public subnet, enabling auto-assign public IP, and creating a security group that allows the required traffic.*

Repeat the same steps for `server2`, but pick the **other** AZ/subnet and use the second User Data script (Server 2 message).

---

### 3. Create a Target Group

A **Target Group** is the object the Load Balancer uses to know *which* backend resources should receive traffic, and *how* to health-check them.

**Target type:**

![Target type selection](images/Screenshot%202026-07-26%20053438.png)
*Four target types are available: `Instances`, `IP addresses`, `Lambda function`, and `Application Load Balancer` (used for ALB-to-ALB chaining). We select **Instances** since we're targeting EC2 instances directly.*

**Protocol, port & VPC:**

![Protocol and port](images/Screenshot%202026-07-26%20053452.png)
*Protocol is `HTTP` on port `80` — this is the port the Load Balancer uses to talk to the backend instances (the "backend port"). The Target Group must be created inside the same VPC as the instances. Protocol version is left at `HTTP1`.*

**Health Check configuration:**

![Health check protocol/path](images/Screenshot%202026-07-26%20053501.png)
*Health check protocol is `HTTP`, and the path is `/` (the root — i.e. `index.html`). AWS periodically sends a request to this path on every registered target to decide if it's healthy.*

![Health check thresholds](images/Screenshot%202026-07-26%20053522.png)
*Fine-tuning the health check behavior:*
- **Healthy threshold = 2** → an instance is marked *healthy* after **2 consecutive successful** checks.
- **Unhealthy threshold = 2** → an instance is marked *unhealthy* after **2 consecutive failed** checks.
- **Timeout = 5 seconds** → how long to wait for a response before considering that particular check failed.
- **Interval = 5 seconds** → how often a health check request is sent.
- **Success codes = 200** → an HTTP `200 OK` response is what counts as "healthy."

> Lower interval/timeout values make the Load Balancer detect failures faster, but generate more health-check traffic on your instances — this is a trade-off between *fast failure detection* and *overhead*.

**Registering targets:**

![Register targets](images/Screenshot%202026-07-26%20053554.png)
*Both instances (`server1` in `us-east-1d`, `server2` in `us-east-1c`) are running and available to be registered into the Target Group. At this point they're marked `unused` because no Load Balancer is routing traffic to them yet.*

---

### 4. Create the Application Load Balancer

**Basic configuration:**

![ALB basic config](images/Screenshot%202026-07-26%20053830.png)
*We name it `ALB-1` and choose **Internet-facing** (since we need to reach it from outside AWS — an `Internal` ALB only gets a private IP and is unreachable from the public internet). IP address type is `IPv4` only.*

**Availability Zones & subnets mapping:**

![AZ and subnet mapping](images/Screenshot%202026-07-26%20053854.png)
*AWS requires **at least two Availability Zones** for an ALB (this is what makes it highly available). We enable `us-east-1c` (mapped to subnet `pub-subnet-az-1c`) and `us-east-1d` (mapped to subnet `pub-subnet-az-1d`) — one subnet per AZ is enough. AWS will place one Load Balancer **node** (with its own ENI and IP) in each selected AZ.*

**Security Groups & Listener:**

![Security group and listener 80](images/Screenshot%202026-07-26%20053911.png)
*Unlike some other Load Balancer types, an **ALB requires a Security Group** of its own. We attach the `default` security group (wide open). The Listener is configured on `HTTP : 80`, with the default action set to **Forward to target groups** → `TG-Test`.*

**Multiple listeners (optional demo):**

![Two listeners 80 and 443](images/Screenshot%202026-07-26%20053951.png)
*An ALB supports multiple listeners — here a second listener on port `443` (using plain HTTP protocol just for demonstration) was added, both forwarding to the same target group. In a real production setup, port `443` should use the `HTTPS` protocol with an SSL/TLS certificate attached, not raw HTTP.*

---

### 5. Test the instances directly (before going through the ALB)

![Server 1 direct test](images/Screenshot%202026-07-26%20054803.png)
*Browsing `server1`'s **public IP** directly shows: "Server 1 — Public Subnet - AZ: us-east-1d — Default VPC". This confirms the User Data script ran successfully and Apache is serving the page.*

Doing the same for `server2`'s public IP returns its own "Server 2 — AZ: us-east-1c" page — confirming both instances are independently reachable before we even introduce the Load Balancer.

---

### 6. Test through the Load Balancer (the real goal)

Once the ALB is `Active` and the Target Group shows both targets as `healthy`, we copy the **ALB's DNS name** (e.g. `alb-1-105746379.us-east-1.elb.amazonaws.com`) and open it in the browser instead of using either instance's IP directly.

![Load Balancer serving Server 1](images/Screenshot%202026-07-26%20054737.png)
*First request through the ALB domain name → routed to `server1` (AZ `us-east-1d`).*

![Load Balancer serving Server 2](images/Screenshot%202026-07-26%20054753.png)
*Refreshing the page → the **same ALB URL** now routes to `server2` (AZ `us-east-1c`). This is the Load Balancer's default **Round Robin** algorithm alternating between healthy targets on every new request.*

Repeated refreshing keeps alternating: Server 1 → Server 2 → Server 1 → Server 2 …

---

## 🌐 Request Flow (Step by Step)

```
1. User's browser requests: http://alb-1-xxxxx.us-east-1.elb.amazonaws.com/
                       │
2. DNS resolution via Route 53 → returns ONE of the ALB node IPs
   (Route 53 knows both node IPs because the ALB registered them
    when it was created across the two selected AZs)
                       │
3. Request hits the ALB node (in AZ 1c OR AZ 1d, whichever IP was returned)
                       │
4. ALB checks the Target Group → picks a HEALTHY target
   using Round Robin (default algorithm)
                       │
5. ALB forwards the request to the chosen EC2 instance on port 80
                       │
6. Apache on that instance returns its index.html page
                       │
7. ALB relays the response back to the user's browser
```

**Why can't we see the ALB node IPs directly?** Because the ALB's IP addresses are dynamic — AWS can add, remove, or replace nodes at any time (e.g. auto-scaling the load balancer itself). That's exactly why we always use the **DNS name**, not a hardcoded IP, to reach an ALB.

---

## 📜 User Data Scripts

Both scripts install Apache and drop a static `index.html` identifying the server. Only the message and background image differ.

### Server 1 — `us-east-1d`

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd

cat <<'EOF' > /var/www/html/index.html
<!DOCTYPE html>
<html>
<head>
  <title>Server 1</title>
  <style>
    body {
      margin: 0;
      height: 100vh;
      display: flex;
      align-items: center;
      justify-content: center;
      background-image: url('https://images.unsplash.com/photo-1518770660439-4636190af475?auto=format&fit=crop&w=1600&q=80');
      background-size: cover;
      background-position: center;
      font-family: Arial, sans-serif;
    }
    .box {
      background: rgba(0,0,0,0.6);
      color: #fff;
      padding: 40px;
      border-radius: 12px;
      text-align: center;
    }
  </style>
</head>
<body>
  <div class="box">
    <h1>Server 1</h1>
    <p>Public Subnet - AZ: us-east-1d</p>
    <p>Default VPC</p>
  </div>
</body>
</html>
EOF
```

### Server 2 — `us-east-1c`

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd

cat <<'EOF' > /var/www/html/index.html
<!DOCTYPE html>
<html>
<head>
  <title>Server 2</title>
  <style>
    body {
      margin: 0;
      height: 100vh;
      display: flex;
      align-items: center;
      justify-content: center;
      background-image: url('https://images.unsplash.com/photo-1451187580459-43490279c0fa?auto=format&fit=crop&w=1600&q=80');
      background-size: cover;
      background-position: center;
      font-family: Arial, sans-serif;
    }
    .box {
      background: rgba(0,0,0,0.6);
      color: #fff;
      padding: 40px;
      border-radius: 12px;
      text-align: center;
    }
  </style>
</head>
<body>
  <div class="box">
    <h1>Server 2</h1>
    <p>Public Subnet - AZ: us-east-1c</p>
    <p>Default VPC</p>
  </div>
</body>
</html>
EOF
```

---

## 📚 Key AWS Concepts Explained

| Concept | What it is | Why it matters here |
|---|---|---|
| **VPC (Virtual Private Cloud)** | An isolated virtual network in AWS. The *Default VPC* spans all AZs in a region. | Lets `server1` and `server2` live in different AZs while remaining on the same private network. |
| **Availability Zone (AZ)** | A physically isolated data center (or group of them) within a region. | Spreading instances across AZs protects against a single data-center failure. |
| **Subnet** | A range of IP addresses within a VPC, tied to *one* AZ. A *public* subnet has a route to an Internet Gateway. | Each instance needs a public subnet to get a public IP and be internet-reachable. |
| **Security Group** | A virtual, stateful firewall attached to instances/ALB. | Controls which ports/IPs are allowed to reach the instances and the ALB. |
| **Target Group** | A logical grouping of backend targets (instances, IPs, or Lambda) plus health-check settings. | The ALB doesn't send traffic directly to instances — it always goes through a Target Group. |
| **Health Check** | Periodic requests the Load Balancer sends to each target to verify it's alive. | Only `healthy` targets receive traffic — this is how the ALB avoids sending users to a broken server. |
| **Application Load Balancer (ALB)** | A Layer 7 (HTTP/HTTPS) load balancer that routes traffic based on content, and distributes it across targets. | Provides a single, stable entry point (DNS name) and high availability across AZs. |
| **Listener** | A process on the Load Balancer that checks for connections on a specific protocol/port. | Defines what to do with incoming traffic (e.g., forward HTTP:80 to `TG-Test`). |
| **Route 53** | AWS's managed DNS service. | Resolves the ALB's DNS name to one of its node IP addresses. |
| **User Data** | A script executed automatically the first time an EC2 instance boots. | Used here to install Apache and generate the identifying HTML page with zero manual SSH work. |

---

## ✅ Best Practices Applied / Recommended

- Use **at least two Availability Zones** for any production Load Balancer (AWS actually *requires* it).
- Keep **Health Check paths lightweight** (root `/` or a dedicated `/health` endpoint) so checks are fast and cheap.
- In production, terminate SSL/TLS at the ALB using an **HTTPS listener (port 443)** with a valid ACM certificate — don't run raw HTTP on 443 as shown in the demo listener.
- Scope the Load Balancer's and instances' **Security Groups** as narrowly as possible in real environments (avoid `0.0.0.0/0` on anything beyond port 80/443).
- Always reference a Load Balancer by its **DNS name**, never by hardcoding one of its node IPs — those IPs can change.
- Tag all resources (ALB, Target Group, instances) for cost tracking and easier cleanup.

---

## ⚠️ Common Mistakes to Avoid

- Forgetting to enable **Auto-assign Public IP** on the EC2 instances → instance becomes unreachable from the internet.
- Placing both instances in the **same subnet/AZ** → defeats the purpose of high availability.
- Registering instances in the Target Group but forgetting the **security group doesn't allow the ALB's traffic** to reach them on the health-check port.
- Trying to add two listeners on the **same port** (e.g., two `HTTP:80` listeners) — AWS will reject this; each listener must have a unique protocol/port combination.
- Testing only via each instance's own public IP and never actually validating the ALB's DNS name — this misses testing the actual load-balancing behavior.

---

## 🧹 Cleanup

To avoid ongoing charges, delete resources in this order:

1. Delete the **Application Load Balancer** (`ALB-1`).
2. Delete the **Target Group** (`TG-Test`).
3. Terminate both **EC2 instances** (`server1`, `server2`).
4. (Optional) Delete any Security Groups created specifically for this lab.

---

## 🏁 Summary

| Component | Purpose | Best Practice |
|---|---|---|
| Default VPC | Shared network across all AZs | Use dedicated VPCs for production workloads |
| Public Subnets (1c / 1d) | Host internet-facing EC2 instances | One subnet per AZ, at least 2 AZs |
| EC2 + User Data | Serve the actual web content | Bake AMIs for faster boot in production instead of long User Data scripts |
| Target Group | Defines backend targets + health checks | Keep health checks fast and specific |
| Application Load Balancer | Distributes traffic, provides single DNS entry point | Enable HTTPS, enable access logs, enable deletion protection |
| Route 53 | DNS resolution for the ALB | Use a custom domain with an alias record to the ALB in production |

---

**Lab result:** Two independent EC2 instances (`server1` in `us-east-1d`, `server2` in `us-east-1c`) inside the Default VPC, both serving through Apache, load-balanced by a single Application Load Balancer with Round Robin distribution — verified end-to-end via direct instance access and via the ALB's public DNS name.