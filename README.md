#  Virtual Private Cloud Management Tool (VPCctl)

This is a Linux-based Virtual Private Cloud (VPC) implementation using network namespaces, bridges, veth pairs, and iptables. This tool recreates AWS-like VPC functionality entirely on a single Linux host.

## Overview

VPCctl is a command-line tool that allows you to:
- Create isolated virtual networks (VPCs) with custom CIDR ranges
- Provision public and private subnets within VPCs
- Enable routing and NAT gateway functionality
- Implement VPC peering for controlled inter-VPC communication
- Apply security groups (firewall rules) to subnets
- Deploy applications (like Nginx) into subnets


## Installation

1. Clone this repository:
```bash
git clone <your-repo-url>
cd vpc-project
```

2. Make the script executable:
```bash
chmod +x vpcctl
```

## Architecture
```
┌─────────────────────────────────────────────────────────┐
│                      Host System                         │
│                                                          │
│  ┌──────────────────────────────────────────────────┐  │
│  │              VPC1 (10.0.1.0/16)                   │  │
│  │                                                   │  │
│  │  ┌────────────────┐      ┌────────────────┐     │  │
│  │  │ Public Subnet  │      │ Private Subnet │     │  │
│  │  │  10.0.1.10/24  │      │  10.0.1.20/24  │     │  │
│  │  │  (Namespace)   │      │  (Namespace)   │     │  │
│  │  │   - Internet   │      │  - No Internet │     │  │
│  │  └───────┬────────┘      └───────┬────────┘     │  │
│  │          │                       │              │  │
│  │          └───────┬───────────────┘              │  │
│  │                  │                              │  │
│  │          ┌───────▼────────┐                     │  │
│  │          │  Bridge (vpc1-br)                    │  │
│  │          │  Gateway: 10.0.1.1                   │  │
│  │          └───────┬────────┘                     │  │
│  └──────────────────┼──────────────────────────────┘  │
│                     │                                  │
│                     │ NAT                              │
│                     ▼                                  │
│              ┌──────────────┐                          │
│              │   Internet   │                          │
│              │   (eth0)     │                          │
│              └──────────────┘                          │
└─────────────────────────────────────────────────────────┘
```

## Usage

### 1. Create a VPC
```bash
./vpcctl create-vpc --name vpc1 --cidr 10.0.1.0/16
```

**Options:**
- `--name`: Unique name for the VPC
- `--cidr`: IP address range in CIDR notation (must include /16 or /24)

### 2. Create Subnets

**Public subnet (with internet access):**
```bash
./vpcctl create-subnet --vpc vpc1 --name public --cidr 10.0.1.10/24 --type public
```

**Private subnet (no internet access):**
```bash
./vpcctl create-subnet --vpc vpc1 --name private --cidr 10.0.1.20/24 --type private
```

**Options:**
- `--vpc`: Name of the parent VPC
- `--name`: Subnet name
- `--cidr`: Subnet IP range
- `--type`: `public` or `private`

### 3. Deploy Applications

Deploy Nginx web server to a subnet:
```bash
./vpcctl deploy-app --vpc vpc1 --subnet public
```

**Options:**
- `--vpc`: VPC name
- `--subnet`: Subnet name
- `--type`: Application type (default: nginx)
- `--port`: Port number (default: 80)

### 4. List Deployed Applications
```bash
./vpcctl list-apps
```

### 5. VPC Peering

**Enable peering between two VPCs:**
```bash
./vpcctl peer-vpc --vpc1 vpc1 --vpc2 vpc2
```

**Remove peering:**
```bash
./vpcctl unpeer-vpc --vpc1 vpc1 --vpc2 vpc2
```

### 6. Security Groups (Firewall Rules)

**Apply security group policy:**
```bash
./vpcctl apply-sg --vpc vpc1 --subnet public --policy sg-web-server.json
```

**List security group rules:**
```bash
./vpcctl list-sg --vpc vpc1 --subnet public
```

**Remove security group:**
```bash
./vpcctl remove-sg --vpc vpc1 --subnet public
```

### 7. Cleanup

**Remove application from subnet:**
```bash
./vpcctl cleanup-app --vpc vpc1 --subnet public
```

**Delete entire VPC:**
```bash
./vpcctl delete-vpc --name vpc1
```

## Complete Example Workflow
```bash
# 1. Create VPC1
./vpcctl create-vpc --name vpc1 --cidr 10.0.1.0/16
./vpcctl create-subnet --vpc vpc1 --name public --cidr 10.0.1.10/24 --type public
./vpcctl create-subnet --vpc vpc1 --name private --cidr 10.0.1.20/24 --type private

# 2. Create VPC2
./vpcctl create-vpc --name vpc2 --cidr 10.0.2.0/16
./vpcctl create-subnet --vpc vpc2 --name public --cidr 10.0.2.10/24 --type public
./vpcctl create-subnet --vpc vpc2 --name private --cidr 10.0.2.20/24 --type private

# 3. Deploy applications
./vpcctl deploy-app --vpc vpc1 --subnet public
./vpcctl deploy-app --vpc vpc1 --subnet private
./vpcctl deploy-app --vpc vpc2 --subnet public
./vpcctl deploy-app --vpc vpc2 --subnet private

# 4. Test isolation (should fail - VPCs are isolated)
sudo ip netns exec vpc1-public ping -c 2 10.0.2.10

# 5. Enable peering
./vpcctl peer-vpc --vpc1 vpc1 --vpc2 vpc2

# 6. Test connectivity (should work now)
sudo ip netns exec vpc1-public ping -c 2 10.0.2.10
sudo ip netns exec vpc1-public curl http://10.0.2.10

# 7. Apply security groups
./vpcctl apply-sg --vpc vpc1 --subnet public --policy sg-web-server.json

# 8. Test from host
curl http://10.0.1.10
curl http://10.0.2.10

# 9. Cleanup
./vpcctl delete-vpc --name vpc1
./vpcctl delete-vpc --name vpc2
```

## How It Works

### Network Namespaces
Each subnet is a separate network namespace, providing isolated network stacks.

### Linux Bridge
Acts as a virtual switch connecting all subnets within a VPC.

### Veth Pairs
Virtual ethernet cables connecting namespaces to the bridge.

### iptables
Provides NAT, isolation, and security group functionality.

### Routing
Static routes enable communication between VPCs when peered.

## Project Structure
```
vpc-project/
├── vpcctl                    # Main CLI tool
├── README.md                 # This file
├── sg-web-server.json        # Example security group policy
├── sg-database.json          # Example security group policy
└── .vpcctl-state/            # Runtime state files (created automatically)
```
