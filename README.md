# Azure Tailscale + Microsoft Entra ID Private Access Lab

## Secure Identity-Based Remote Access to Private Azure Workloads

This project demonstrates a production-style remote-access architecture using **Microsoft Entra ID**, **Tailscale**, and **Azure hub-and-spoke networking**.

The objective is to allow an authenticated employee to securely access a **private Azure VM with no public IP address** from a remote workstation.

Instead of exposing the workload directly to the Internet, a Tailscale subnet router is deployed in an Azure Hub VNet. The router advertises the Azure Spoke network to authorized Tailscale clients.

Access is controlled at multiple layers:

- Microsoft Entra ID authentication
- Entra security-group membership
- Tailscale user/device authentication
- Tailscale access policies
- Tailscale subnet routing
- Azure VNet peering
- Azure Network Security Groups
- SSH authentication

---

# Architecture

```text
Remote Windows Workstation
User: Daniel Reed
Microsoft Entra ID
Tailscale IP: 100.98.140.79
        |
        | Encrypted Tailscale / WireGuard tunnel
        |
        v
+--------------------------------------+
| Azure Hub VNet                       |
| vnet-hub-tailscale                   |
| 10.10.0.0/16                         |
|                                      |
| snet-tailscale                       |
| 10.10.1.0/24                         |
|                                      |
| ts-router-01                         |
| Azure IP: 10.10.1.4                  |
| Tailscale IP: 100.67.204.3           |
|                                      |
| Tailscale Subnet Router              |
| Linux IP Forwarding Enabled          |
| Azure NIC IP Forwarding Enabled      |
+------------------+-------------------+
                   |
                   | Azure VNet Peering
                   |
                   v
+--------------------------------------+
| Azure Spoke VNet                     |
| vnet-spoke-workload                  |
| 10.20.0.0/16                         |
|                                      |
| snet-workload                        |
| 10.20.1.0/24                         |
|                                      |
| app-vm-01                            |
| Private IP: 10.20.1.4                |
| Public IP: NONE                      |
+--------------------------------------+
```

Final management path:

```text
Daniel's Windows Workstation
        |
        | Microsoft Entra authentication
        v
Tailscale Client
100.98.140.79
        |
        | Encrypted tunnel
        v
Tailscale Subnet Router
100.67.204.3
10.10.1.4
        |
        | IP forwarding / subnet routing
        v
Azure Hub VNet
        |
        | VNet Peering
        v
Azure Spoke VNet
        |
        | NSG
        v
app-vm-01
10.20.1.4
        |
        +--> ICMP
        |
        +--> SSH TCP/22
```

---

# Lab Objectives

By completing this lab, the environment will provide:

1. Microsoft Entra-based user authentication
2. A dedicated Tailscale access security group
3. A Tailscale organization/tailnet
4. Separate administrator and standard-user roles
5. Azure hub-and-spoke networking
6. A Linux-based Tailscale subnet router
7. Azure VNet peering
8. Azure NIC IP forwarding
9. Linux kernel IP forwarding
10. Persistent Linux forwarding configuration
11. Tailscale subnet-route advertisement
12. A private Azure workload VM
13. No public IP on the protected workload
14. NSG-based workload protection
15. Least-privilege Tailscale access rules
16. Windows Tailscale client connectivity
17. Private ICMP connectivity
18. Private SSH management

---

# Addressing Plan

| Component | Address |
|---|---|
| Hub VNet | `10.10.0.0/16` |
| Tailscale Router Subnet | `10.10.1.0/24` |
| Tailscale Router Azure IP | `10.10.1.4` |
| Tailscale Router Tailscale IP | `100.67.204.3` |
| Spoke VNet | `10.20.0.0/16` |
| Workload Subnet | `10.20.1.0/24` |
| Private Application VM | `10.20.1.4` |
| Daniel's Tailscale Client | `100.98.140.79` |

---

# Prerequisites

The following are required:

- Microsoft Azure subscription
- Microsoft Entra tenant
- Permission to create users/groups
- Tailscale account
- Windows workstation
- Azure Linux VM permissions
- SSH client
- Basic Azure networking knowledge

The lab uses Ubuntu Linux for the Azure virtual machines.

---

# Step 1 — Create the Microsoft Entra Security Group

Open:

**Microsoft Entra admin center → Identity → Groups → All groups**

Create a security group for users who should receive Tailscale access.

Example:

```text
Group Type: Security
Group Name: Tailscale-Users
Membership Type: Assigned
```

The purpose of this group is to establish an identity boundary for remote-access users.

> In a production deployment using supported identity provisioning, Entra groups can be used as part of automated user lifecycle and application assignment. In this lab, some Tailscale membership operations are performed manually because of licensing/lab constraints.

### Screenshot

![Entra Tailscale Security Group](screenshots/01-entra-tailscale-security-group.png)

---

# Step 2 — Add Daniel to the Entra Group

Open:

**Entra ID → Groups → Tailscale-Users → Members**

Add:

```text
Daniel Reed
```

Verify that Daniel appears as a member.

### Screenshot

![Entra Group Membership](screenshots/02-entra-tailscale-group-membership.png)

---

# Step 3 — Configure the Tailscale Organization

Create or sign in to the Tailscale organization using the Microsoft Entra tenant identity.

During this lab, the Tailscale organization became associated with the Entra tenant domain.

Two identities were used:

```text
Adminbasit@basitiqra2023gmail.onmicrosoft.com
daniel.reed@basitiqra2023gmail.onmicrosoft.com
```

The administrator account owns the tailnet while Daniel represents a normal employee.

---

# Step 4 — Configure Tailscale Roles

Open:

**Tailscale Admin Console → Users**

Configure the identities as:

```text
Adminbasit
Role: Owner

Daniel Reed
Role: Member
```

Daniel should not require administrative access to Tailscale simply to reach an authorized workload.

This follows least-privilege principles.

### Screenshot

![Tailscale Users](screenshots/03-tailscale-users-entra-identities.png)

---

# Step 5 — Understand Production Identity Provisioning

In this lab, Daniel was admitted into the Tailscale organization through the available lab/invite workflow.

A larger organization would normally integrate identity lifecycle more deeply with its identity provider where the organization's Tailscale plan and configuration support it.

Conceptually:

```text
Employee
   |
   v
Microsoft Entra ID
   |
   +--> Group / Application Assignment
   |
   v
Tailscale
```

This is important operationally.

When an employee joins, changes roles, or leaves the organization, access should ideally be driven by the identity system rather than maintained independently in many applications.

The exact provisioning/deprovisioning capabilities depend on the Tailscale plan and identity configuration in use.

---

# Step 6 — Create the Azure Resource Group

Create:

```text
rg-tailscale-entra-lab
```

All Azure resources for the lab can be placed inside this resource group.

### Screenshot

![Azure Resource Group](screenshots/04-azure-resource-group-overview.png)

---

# Step 7 — Create the Hub VNet

Create:

```text
Name:
vnet-hub-tailscale

Address Space:
10.10.0.0/16
```

Create the Tailscale subnet:

```text
Name:
snet-tailscale

CIDR:
10.10.1.0/24
```

The Tailscale subnet router will reside here.

Architecture:

```text
vnet-hub-tailscale
10.10.0.0/16

└── snet-tailscale
    10.10.1.0/24

    └── ts-router-01
```

### Screenshot

![Hub VNet](screenshots/05-hub-vnet-tailscale-subnet.png)

---

# Step 8 — Create the Spoke VNet

Create:

```text
Name:
vnet-spoke-workload

Address Space:
10.20.0.0/16
```

Create:

```text
Name:
snet-workload

CIDR:
10.20.1.0/24
```

Architecture:

```text
vnet-spoke-workload
10.20.0.0/16

└── snet-workload
    10.20.1.0/24

    └── app-vm-01
        10.20.1.4
```

### Screenshot

![Spoke VNet](screenshots/06-spoke-vnet-workload-subnet.png)

---

# Step 9 — Peer the Hub and Spoke VNets

Open:

**vnet-hub-tailscale → Peerings → Add**

Create:

```text
hub-to-spoke
```

Configure the reverse peering as well if the Azure portal does not automatically create both directions.

Verify:

```text
Peering State: Connected
Peering Sync Status: Fully Synchronized
```

The hub and spoke can now exchange traffic using Azure private networking.

```text
10.10.0.0/16
     |
     | VNet Peering
     |
10.20.0.0/16
```

### Screenshot

![Hub Spoke Peering](screenshots/07-hub-spoke-vnet-peering.png)

---

# Step 10 — Create the Tailscale Router VM

Deploy an Ubuntu VM.

Example:

```text
Name:
ts-router-01

VNet:
vnet-hub-tailscale

Subnet:
snet-tailscale

Private IP:
10.10.1.4
```

A temporary public IP may be used during initial bootstrap if necessary.

For a more hardened production deployment, administrative access should use a private management path instead of leaving SSH exposed to the Internet.

### Screenshot

![Tailscale Router](screenshots/08-tailscale-router-vm-overview.png)

---

# Step 11 — Protect Router SSH with an NSG

During bootstrap, if a public IP is used, do not allow SSH from the entire Internet.

Example inbound rule:

```text
Source:
<administrator-public-IP>/32

Destination:
Any

Service:
SSH

Protocol:
TCP

Destination Port:
22

Action:
Allow
```

Example:

```text
67.x.x.x/32 → TCP/22 → Allow
```

This limits public SSH access to the administrator's current public IP.

Once private administrative access is available, the public IP can be removed if it is no longer required.

---

# Step 12 — Enable Azure NIC IP Forwarding

A normal VM NIC is generally expected to receive traffic addressed to that VM.

Our Tailscale router is different.

It must forward traffic between:

```text
Tailscale
      ↓
ts-router-01
      ↓
Azure networks
```

Therefore enable IP forwarding on the Azure NIC attached to `ts-router-01`.

Open:

**Azure → ts-router-01 → Networking → Network Interface**

Enable:

```text
IP forwarding: Enabled
```

### Screenshot

![Azure NIC Forwarding](screenshots/09-router-nic-ip-forwarding-enabled.png)

---

# Step 13 — Enable Linux Kernel IP Forwarding

Azure allowing forwarding at the NIC is only one half of the configuration.

Linux itself must also be allowed to route packets.

Check the current setting:

```bash
sysctl net.ipv4.ip_forward
```

Initially this may return:

```text
net.ipv4.ip_forward = 0
```

Enable forwarding:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Verify:

```bash
sysctl net.ipv4.ip_forward
```

Expected:

```text
net.ipv4.ip_forward = 1
```

---

# Step 14 — Make Linux Forwarding Persistent

The previous `sysctl -w` change affects the running system.

Create a persistent configuration:

```bash
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-tailscale.conf
```

Apply the configuration:

```bash
sudo sysctl --system
```

Verify again:

```bash
sysctl net.ipv4.ip_forward
```

Expected:

```text
net.ipv4.ip_forward = 1
```

This ensures forwarding remains enabled after reboot.

---

# Step 15 — Install Tailscale

Install Tailscale on `ts-router-01`.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

What this pipeline does:

```text
curl
  |
  | downloads install.sh
  v
stdout
  |
  | pipe |
  v
sh
  |
  v
executes the installation script
```

Options:

```text
-f    Fail on HTTP errors
-s    Silent
-S    Still display errors
-L    Follow redirects
```

Verify:

```bash
tailscale version
```

---

# Step 16 — Authenticate the Router

Run:

```bash
sudo tailscale up
```

Tailscale returns an authentication URL.

Open the URL and authenticate using the authorized administrator identity.

After successful authentication:

```bash
tailscale status
```

The machine should appear in the tailnet.

### Screenshot

![Tailscale Authentication](screenshots/10-tailscale-router-authentication.png)

---

# Step 17 — Configure the VM as a Tailscale Subnet Router

The spoke network is:

```text
10.20.0.0/16
```

Advertise that route:

```bash
sudo tailscale up --advertise-routes=10.20.0.0/16
```

If the node is already configured and Tailscale asks you to specify existing non-default settings, inspect the current preferences and use the appropriate `tailscale set` or complete `tailscale up` command for that installation.

The conceptual result must be:

```text
ts-router-01
     |
     +---- advertises ----> 10.20.0.0/16
```

---

# Step 18 — Approve the Advertised Route

Open:

**Tailscale Admin Console → Machines → ts-router-01**

Find the subnet route:

```text
10.20.0.0/16
```

Approve it.

Expected:

```text
Approved:
10.20.0.0/16
```

### Screenshot

![Approved Subnet Route](screenshots/11-tailscale-approved-spoke-route.png)

Advertising and authorization are different concepts.

The subnet router tells Tailscale:

> I know how to reach 10.20.0.0/16.

That does **not** mean every user should automatically be authorized to access every address in that network.

Access policy handles that separately.

---

# Step 19 — Create the Private Spoke Workload VM

Deploy another Ubuntu VM.

Configure:

```text
Name:
app-vm-01

VNet:
vnet-spoke-workload

Subnet:
snet-workload

Private IP:
10.20.1.4
```

The protected workload should ultimately have:

```text
Public IP:
NONE
```

During the lab a public IP was accidentally associated with the VM and was subsequently disassociated.

That actually provided an important security validation: the final workload should be reachable through the private access architecture rather than through a public interface.

### Screenshot

![Private Spoke VM](screenshots/12-private-spoke-vm.png)

---

# Step 20 — Configure the Spoke VM NSG

Attach an NSG to the workload NIC/subnet as appropriate.

For this lab, allow management/testing traffic from the Tailscale subnet router's Azure private IP:

```text
10.10.1.4/32
```

SSH rule:

```text
Name:
Allow-SSH

Source:
10.10.1.4/32

Protocol:
TCP

Destination Port:
22

Action:
Allow
```

ICMP rule:

```text
Name:
Allow-ICMP

Source:
10.10.1.4/32

Protocol:
ICMP

Action:
Allow
```

The intended security model is:

```text
Internet
   X
   |
   |
app-vm-01


Tailscale Router
10.10.1.4
   |
   | ALLOWED
   v
app-vm-01
10.20.1.4
```

### Screenshot

![Spoke NSG Rules](screenshots/13-spoke-vm-nsg-tailscale-rules.png)

---

# Step 21 — Understand Tailscale Subnet-Router SNAT

By default, Tailscale subnet routing commonly uses source NAT for traffic forwarded from Tailscale clients into the advertised subnet.

The original client may have a Tailscale address such as:

```text
100.98.140.79
```

But the Azure workload sees the connection arriving through the subnet router.

For this lab, that means the Azure-side source used for the workload security rule is:

```text
10.10.1.4
```

Conceptually:

```text
Daniel
100.98.140.79
      |
      | Tailscale
      v
ts-router-01
100.67.204.3
10.10.1.4
      |
      | SNAT
      v
Source becomes:
10.10.1.4
      |
      v
app-vm-01
10.20.1.4
```

This is why the workload NSG can permit:

```text
Source: 10.10.1.4/32
```

instead of permitting arbitrary Internet addresses.

SNAT also simplifies the return path because the workload replies to `10.10.1.4`, which is directly reachable through Azure networking. The subnet router then maintains the connection state and sends the response back through Tailscale to the originating client.

---

# Step 22 — Configure Least-Privilege Tailscale Access

Advertising:

```text
10.20.0.0/16
```

does **not** require granting Daniel access to every resource in that network.

Our actual workload is:

```text
10.20.1.4
```

Therefore restrict Daniel's access to:

```text
10.20.1.4/32
```

Allow only the protocols required for the lab:

```text
ICMP
TCP/22
```

Conceptually:

```text
Daniel
   |
   +---- ICMP ----+
   |              |
   +---- TCP/22 --+----> 10.20.1.4
```

Not:

```text
Daniel ---> 10.20.0.0/16 : ANY
```

### Screenshot

![Tailscale Least Privilege Grant](screenshots/14-tailscale-daniel-least-privilege-grant.png)

---

# Step 23 — Install Tailscale on the Employee Workstation

Install the Tailscale client on the Windows workstation.

Launch Tailscale and authenticate using:

```text
Daniel Reed
```

through Microsoft Entra ID.

After successful authentication, the workstation joins the tailnet.

For this lab:

```text
Windows Tailscale IP:
100.98.140.79
```

### Screenshot

![Daniel Tailscale Connected](screenshots/15-daniel-windows-tailscale-connected.png)

---

# Step 24 — Verify Tailscale Status

From Windows PowerShell:

```powershell
tailscale status
```

Example:

```text
100.98.140.79   syedbasit26    daniel.reed@    windows
100.67.204.3    ts-router-01   Adminbasit@     linux
```

This proves both endpoints participate in the same Tailscale environment.

---

# Step 25 — Verify the Windows Route

Run:

```powershell
route print
```

The workstation should contain a route for:

```text
10.20.0.0
255.255.0.0
```

In this lab the route appeared similar to:

```text
Network Destination    Netmask          Gateway          Interface
10.20.0.0              255.255.0.0      100.100.100.100  100.98.140.79
```

Interpretation:

```text
Destination:
10.20.0.0/16
        |
        v
Send into Tailscale
        |
        v
Local Tailscale Interface:
100.98.140.79
```

`100.98.140.79` is Daniel's local Windows Tailscale interface.

It is **not** the remote Azure router.

The Tailscale client and operating system cooperate to steer traffic for the advertised subnet into the Tailscale networking stack.

### Screenshot

![Windows Tailscale Route](screenshots/16-windows-tailscale-spoke-route.png)

---

# Step 26 — Test Private ICMP Connectivity

From Daniel's Windows workstation:

```powershell
ping 10.20.1.4
```

Successful result:

```text
Reply from 10.20.1.4
Reply from 10.20.1.4
Reply from 10.20.1.4
Reply from 10.20.1.4

Packets: Sent = 4, Received = 4, Lost = 0
```

The packet is effectively traversing:

```text
Windows
   |
   v
Tailscale Client
   |
   v
Encrypted Tailscale Path
   |
   v
ts-router-01
   |
   v
Hub VNet
   |
   v
VNet Peering
   |
   v
Spoke VNet
   |
   v
NSG
   |
   v
10.20.1.4
```

### Screenshot

![Private Ping Success](screenshots/17-tailscale-private-vm-ping-success.png)

---

# Step 27 — Test Private SSH Access

Now perform a real management test rather than relying only on ICMP.

From Windows:

```powershell
ssh azureuser@10.20.1.4
```

After authentication:

```bash
whoami
hostname
ip -4 addr show eth0
```

Expected:

```text
azureuser

app-vm-01

inet 10.20.1.4/24
```

This confirms that Daniel successfully established an SSH session to an Azure VM that has **no public IP address**.

### Screenshot

![Private SSH Success](screenshots/18-tailscale-private-vm-ssh-success.png)

---

# Step 28 — End-to-End Packet Flow

The complete packet flow is:

```text
1. Daniel authenticates using Microsoft Entra ID

2. Daniel's Windows machine joins the Tailscale network

3. Tailscale installs/handles the route for:
   10.20.0.0/16

4. Daniel initiates:
   ssh azureuser@10.20.1.4

5. Windows recognizes 10.20.1.4 belongs to:
   10.20.0.0/16

6. Traffic enters the Tailscale interface.

7. Tailscale encrypts the traffic.

8. Traffic reaches:
   ts-router-01

9. The subnet router forwards the packet toward Azure.

10. Linux IP forwarding permits routing.

11. Azure NIC IP forwarding permits forwarded traffic.

12. The router forwards toward the spoke network.

13. Azure VNet peering provides connectivity between:
    10.10.0.0/16
    and
    10.20.0.0/16

14. The workload NSG evaluates the connection.

15. Traffic from the subnet router is allowed to:
    TCP/22
    and ICMP

16. app-vm-01 receives the connection.

17. Return traffic reaches the subnet router.

18. Tailscale returns the encrypted traffic to Daniel.
```

---

# Step 29 — Security Layers

This architecture does not depend on a single control.

## Layer 1 — Identity

Microsoft Entra ID authenticates the user.

```text
Who are you?
```

---

## Layer 2 — Tailscale Membership

The user/device must participate in the appropriate tailnet.

```text
Are you part of this private network?
```

---

## Layer 3 — Tailscale Access Policy

Daniel is authorized only for the required destination/protocols.

```text
What private resource may you access?
```

---

## Layer 4 — Tailscale Routing

The subnet router advertises reachability to:

```text
10.20.0.0/16
```

```text
How do I reach that Azure network?
```

---

## Layer 5 — Azure Network Security Group

The workload accepts the lab's management traffic only from:

```text
10.10.1.4/32
```

```text
Which Azure-side source and protocol may reach this VM?
```

---

## Layer 6 — Host Authentication

SSH still requires valid operating-system authentication.

```text
Are you authorized to log into this Linux host?
```

---

# Step 30 — Why Advertise /16 but Authorize /32?

The router advertises:

```text
10.20.0.0/16
```

because it provides connectivity to the Azure spoke network.

But Daniel is authorized only to:

```text
10.20.1.4/32
```

These serve different purposes.

```text
ROUTING
"Where can traffic potentially be delivered?"

10.20.0.0/16
```

versus:

```text
AUTHORIZATION
"What is this user actually allowed to access?"

10.20.1.4/32
```

This separation is extremely important.

A route being known does not mean access should automatically be granted.

---

# Step 31 — Why the Workload Does Not Need a Public IP

Without private remote access, administrators sometimes expose workloads through:

```text
Public IP
   |
Internet
   |
SSH
```

Our final architecture instead provides:

```text
Employee
   |
Entra Authentication
   |
Tailscale
   |
Private Subnet Router
   |
Azure Private Networking
   |
Private VM
```

Therefore:

```text
app-vm-01
Public IP: NONE
Private IP: 10.20.1.4
```

The workload does not need Internet-facing SSH.

---

# Step 32 — Why Both Azure and Linux Forwarding Are Required

Two separate forwarding controls were configured.

## Azure NIC

```text
IP Forwarding = Enabled
```

This allows the Azure NIC to participate in forwarding traffic that is not simply destined for the router VM itself.

## Linux

```text
net.ipv4.ip_forward = 1
```

This tells the Linux kernel to route IPv4 packets between interfaces/paths.

Think of it as:

```text
Azure says:
"This NIC may forward packets."

Linux says:
"This operating system will forward packets."
```

Both must align for the VM to function properly as a network virtual appliance/subnet router.

---

# Step 33 — Troubleshooting Commands

## Check Tailscale

Linux:

```bash
tailscale status
```

Windows:

```powershell
tailscale status
```

---

## Check Linux forwarding

```bash
sysctl net.ipv4.ip_forward
```

Expected:

```text
net.ipv4.ip_forward = 1
```

---

## Check Linux routes

```bash
ip route
```

---

## Check interfaces

```bash
ip addr
```

---

## Check Windows routes

```powershell
route print
```

---

## Test ICMP

```powershell
ping 10.20.1.4
```

---

## Test TCP/22

```powershell
Test-NetConnection 10.20.1.4 -Port 22
```

---

## Test SSH

```powershell
ssh azureuser@10.20.1.4
```

---

# Step 34 — Troubleshooting Flow

If the private VM cannot be reached, troubleshoot in layers instead of randomly changing settings.

```text
1. Is Tailscale connected?
        |
        v
2. Is Daniel authenticated?
        |
        v
3. Does tailscale status show the router?
        |
        v
4. Is 10.20.0.0/16 advertised?
        |
        v
5. Is the subnet route approved?
        |
        v
6. Does the Tailscale access policy permit the traffic?
        |
        v
7. Is Linux IP forwarding enabled?
        |
        v
8. Is Azure NIC IP forwarding enabled?
        |
        v
9. Is Hub-Spoke peering Connected?
        |
        v
10. Does the workload NSG permit the router source?
        |
        v
11. Is SSH running on app-vm-01?
        |
        v
12. Test connectivity again.
```

Useful Linux SSH check:

```bash
sudo systemctl status ssh
```

Listening ports:

```bash
sudo ss -tulpn
```

---

# Step 35 — Important Lesson: Authentication vs Authorization vs Routing

This lab demonstrates three concepts that are often confused.

## Authentication

```text
Microsoft Entra ID

"Who is Daniel?"
```

## Authorization

```text
Tailscale Policy + Azure NSG

"What is Daniel allowed to access?"
```

## Routing

```text
Tailscale Subnet Router + Azure VNet Peering

"How does the packet physically/logically reach 10.20.1.4?"
```

Successful private access requires all three.

---

# Step 36 — Comparison with Traditional VPN Architecture

A traditional Azure remote-access design may use components such as:

```text
Remote User
     |
     v
VPN Gateway / Firewall
     |
     v
Hub
     |
     v
Spoke
```

This lab demonstrates another approach:

```text
Remote User
     |
     v
Entra + Tailscale
     |
     v
Tailscale Subnet Router
     |
     v
Hub
     |
     v
Spoke
```

The correct production architecture depends on organizational requirements including:

- security inspection requirements
- compliance
- user/device scale
- logging requirements
- identity integration
- high availability
- existing firewall architecture
- site-to-site connectivity
- operational requirements

Tailscale does not automatically replace every function provided by an enterprise firewall or traditional VPN architecture.

---

# Step 37 — Production Improvements

This lab intentionally focuses on understanding the networking and identity path.

A production implementation could improve it further with:

- Multiple subnet routers for high availability
- Automated identity provisioning/deprovisioning
- Device posture checks
- Stronger device-management requirements
- Conditional Access where supported by the identity/application design
- Centralized logging
- Azure Monitor
- Microsoft Sentinel
- Removal of bootstrap public IPs
- Private administration
- Infrastructure as Code
- Terraform
- Automated policy deployment
- Multiple workload tiers
- Separate production/non-production networks
- Dedicated firewall/NVA inspection where required
- Formal Joiner-Mover-Leaver processes

---

# Step 38 — Cleanup

To avoid unnecessary Azure charges, remove resources when the lab is complete.

If the entire resource group exists only for this project:

```text
rg-tailscale-entra-lab
```

it can be deleted after documentation is complete.

Also review:

- VM disks
- Public IP addresses
- NICs
- NSGs
- VNets
- Tailscale devices
- Tailscale subnet routes
- Test users/groups if no longer required

Never delete shared identity or networking resources without verifying that other workloads do not depend on them.

---

# Final Validation

The lab was considered successful after all of the following were verified:

```text
[✓] Microsoft Entra user created
[✓] Tailscale security group configured
[✓] Daniel added to the group
[✓] Tailscale organization configured
[✓] Administrator/owner identity established
[✓] Daniel configured as standard member
[✓] Azure Hub VNet created
[✓] Azure Spoke VNet created
[✓] Hub/Spoke peering connected
[✓] Tailscale router deployed in Hub
[✓] Azure NIC forwarding enabled
[✓] Linux IPv4 forwarding enabled
[✓] Linux forwarding made persistent
[✓] Tailscale installed on router
[✓] Spoke subnet advertised
[✓] Advertised route approved
[✓] Private workload VM deployed
[✓] Workload public IP removed
[✓] Workload NSG restricted
[✓] Tailscale least-privilege policy configured
[✓] Windows Tailscale client authenticated as Daniel
[✓] 10.20.0.0/16 route visible on workstation
[✓] ICMP to 10.20.1.4 successful
[✓] SSH to 10.20.1.4 successful
```

---

# Final Result

The completed environment provides the following access path:

```text
Microsoft Entra ID
        |
        v
Daniel Reed
        |
        v
Tailscale Windows Client
100.98.140.79
        |
        | Encrypted private connectivity
        v
ts-router-01
100.67.204.3
10.10.1.4
        |
        | Azure Hub
        v
VNet Peering
        |
        | Azure Spoke
        v
NSG
        |
        v
app-vm-01
10.20.1.4
NO PUBLIC IP
        |
        +---- ICMP SUCCESS
        |
        +---- SSH SUCCESS
```

The key outcome is that an Entra-authenticated remote employee can securely administer a **private Azure workload without exposing that workload directly to the public Internet**.

---

# Screenshots

The repository contains the following implementation evidence:

```text
screenshots/
├── 01-entra-tailscale-security-group.png
├── 02-entra-tailscale-group-membership.png
├── 03-tailscale-users-entra-identities.png
├── 04-azure-resource-group-overview.png
├── 05-hub-vnet-tailscale-subnet.png
├── 06-spoke-vnet-workload-subnet.png
├── 07-hub-spoke-vnet-peering.png
├── 08-tailscale-router-vm-overview.png
├── 09-router-nic-ip-forwarding-enabled.png
├── 10-tailscale-router-authentication.png
├── 11-tailscale-approved-spoke-route.png
├── 12-private-spoke-vm.png
├── 13-spoke-vm-nsg-tailscale-rules.png
├── 14-tailscale-daniel-least-privilege-grant.png
├── 15-daniel-windows-tailscale-connected.png
├── 16-windows-tailscale-spoke-route.png
├── 17-tailscale-private-vm-ping-success.png
└── 18-tailscale-private-vm-ssh-success.png
```

---

## Skills Practiced

This project provided hands-on experience with:

**Azure Networking**
- Virtual Networks
- Subnets
- Hub-and-spoke architecture
- VNet peering
- Network Security Groups
- Private IP addressing
- Azure NIC forwarding

**Linux Networking**
- IPv4 forwarding
- `sysctl`
- Persistent kernel configuration
- Routing
- SSH

**Microsoft Entra ID**
- Users
- Security groups
- Identity-based authentication
- Access lifecycle concepts

**Tailscale**
- Tailnets
- Device enrollment
- User roles
- Subnet routers
- Route advertisement
- Route approval
- Access policies/grants
- Private overlay networking

**Security**
- Least privilege
- Identity-based access
- Private workload access
- Removal of unnecessary public exposure
- Layered network controls
- Authentication vs authorization
- Defense in depth

---

## Project Summary

This project demonstrates how modern identity-aware networking can provide secure remote access to private cloud workloads.

Rather than exposing the application VM to the Internet, the design combines **Microsoft Entra ID for identity**, **Tailscale for encrypted remote connectivity and policy**, and **Azure hub-and-spoke networking plus NSGs for private workload isolation**.

The final validation successfully connected an Entra-authenticated Windows user to `app-vm-01` at the private address `10.20.1.4` using both ICMP and SSH.
