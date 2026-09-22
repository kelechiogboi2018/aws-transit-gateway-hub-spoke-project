# AWS Transit Gateway Hub-and-Spoke Networking Project

## Project Overview

This project demonstrates the design, configuration, validation, and
troubleshooting of a multi-VPC AWS network using **AWS Transit Gateway
(TGW)** as a central hub.

Three VPC environments were connected through the Transit Gateway.
Amazon Linux EC2 instances in separate VPCs were then used to verify
private connectivity with **ICMP (ping)** and **SSH (TCP/22)**.

The project also demonstrates practical troubleshooting of AWS routing,
Security Groups, private/public IP addressing, and SSH agent forwarding.

## Architecture

``` text
                         Windows Workstation
                                |
                         SSH + Agent Forwarding
                                |
                                v
                     EC2-A / Entry Instance
                         10.0.1.29
                                |
                                v
                     +------------------+
                     |  Transit Gateway |
                     |       HUB        |
                     +------------------+
                        /             \
                       /               \
                      v                 v
               VPC-B / Spoke       VPC-C / Spoke
                10.1.0.0/16         10.2.0.0/16
                      |                   |
                      v                   v
                   EC2-B               EC2-C
                 10.1.2.44          10.2.2.173
```

  Environment   CIDR            EC2 private IP
  ------------- --------------- ----------------
  VPC-A         `10.0.0.0/16`   `10.0.1.29`
  VPC-B         `10.1.0.0/16`   `10.1.2.44`
  VPC-C         `10.2.0.0/16`   `10.2.2.173`

> Public IP addresses are omitted because they are not required to
> explain the architecture and may change.

## Project Objectives

-   Build three isolated VPC networks with non-overlapping CIDRs.
-   Connect them through AWS Transit Gateway.
-   Configure VPC and TGW routing.
-   Deploy EC2 instances for testing.
-   Verify VPC-to-VPC communication over private IP addresses.
-   Configure Security Groups for ICMP and SSH.
-   Use SSH agent forwarding instead of copying private keys to
    intermediate servers.
-   Troubleshoot realistic AWS networking and SSH failures.

## AWS Services and Technologies

-   Amazon VPC
-   AWS Transit Gateway
-   Amazon EC2
-   Subnets and VPC Route Tables
-   Transit Gateway VPC Attachments and routing
-   Internet Gateway for the externally reachable entry instance
-   Security Groups
-   Amazon Linux 2023
-   SSH / SSH Agent Forwarding
-   ICMP and TCP/IP
-   PowerShell / OpenSSH

## Implementation

### 1. VPC Design

Three non-overlapping VPC CIDR ranges were used:

``` text
VPC-A: 10.0.0.0/16
VPC-B: 10.1.0.0/16
VPC-C: 10.2.0.0/16
```

Non-overlapping CIDRs allow AWS routing to identify each destination
network unambiguously.

### 2. Subnets and Route Tables

Subnets were created inside the VPCs and associated with their route
tables. Routes were configured so traffic for remote VPC CIDRs could be
forwarded toward the Transit Gateway.

Conceptually:

``` text
VPC-A -> TGW -> VPC-B
VPC-A -> TGW -> VPC-C
VPC-B -> TGW -> VPC-C
```

Return routing is equally important: successful communication requires a
valid path in both directions.

### 3. Transit Gateway

AWS Transit Gateway was configured as the central networking hub, with
the VPCs attached as spokes.

``` text
                 TGW
              /   |   \
             /    |    \
          VPC-A VPC-B VPC-C
```

This avoids building a growing mesh of individual point-to-point VPC
connections.

### 4. EC2 Test Instances

Amazon Linux 2023 EC2 instances were used as endpoints:

``` text
EC2-A: 10.0.1.29
EC2-B: 10.1.2.44
EC2-C: 10.2.2.173
```

EC2-A acted as the initial SSH entry point.

## Connectivity Testing

### EC2-A to EC2-B

``` bash
ping -c 4 10.1.2.44
```

Result: **4 transmitted, 4 received, 0% packet loss**.

### EC2-A to EC2-C

``` bash
ping -c 4 10.2.2.173
```

Result: **4 transmitted, 4 received, 0% packet loss**.

### EC2-B to EC2-C

The initial test failed:

``` bash
ping -c 4 10.2.2.173
```

Result: **4 transmitted, 0 received, 100% packet loss**.

EC2-C's Security Group allowed ICMP from VPC-A (`10.0.0.0/16`) but not
VPC-B (`10.1.0.0/16`). After adding the required ICMP rule for VPC-B,
the same test returned **4 transmitted, 4 received, 0% packet loss**.

This demonstrated spoke-to-spoke communication through the Transit
Gateway.

## SSH Troubleshooting

### Connection Timed Out

An SSH attempt produced:

``` text
ssh: connect to host <address> port 22: Connection timed out
```

The investigation included Security Group inbound rules, source CIDRs,
private versus public IP selection, VPC/TGW routing, and TCP port 22.

After allowing SSH from the correct source network, the error changed
to:

``` text
Permission denied (publickey,...)
```

This was useful evidence: the network path and TCP/22 were now working,
leaving authentication as the remaining problem.

### SSH Agent Forwarding

Rather than copying the `.pem` private key to EC2-A or EC2-B, SSH agent
forwarding was used.

From Windows:

``` powershell
ssh-add -l
ssh -A -i "Devops-TGW-London-Key-Key.pem" ec2-user@<EC2-A-PUBLIC-IP>
```

On EC2-A:

``` bash
ssh-add -l
ssh ec2-user@10.1.2.44
```

To forward the agent onward through EC2-B:

``` bash
ssh -A ec2-user@10.1.2.44
```

On EC2-B:

``` bash
ssh-add -l
ssh ec2-user@10.2.2.173
```

The final prompt confirmed successful access to EC2-C:

``` text
[ec2-user@ip-10-2-2-173 ~]$
```

The completed path was:

``` text
Windows Workstation
        |
        | SSH -A
        v
EC2-A (10.0.1.29)
        |
        | SSH -A
        v
EC2-B (10.1.2.44)
        |
        | SSH
        v
EC2-C (10.2.2.173)
```

No private key was copied onto the intermediate EC2 instances.

## Troubleshooting Summary

  ---------------------------------------------------------------------------------
  Problem                 Observation                       Resolution
  ----------------------- --------------------------------- -----------------------
  Laptop could not SSH to Port 22 timed out                 Updated SSH inbound
  EC2-A                                                     source to the current
                                                            trusted client IP

  EC2-A could ping EC2-B  ICMP worked; TCP/22 timed out     Allowed TCP/22 from the
  but SSH failed                                            required VPC-A source

  EC2-A reached EC2-B but `Permission denied (publickey)`   Enabled SSH agent
  authentication failed                                     forwarding

  EC2-B could not ping    100% packet loss                  Allowed ICMP from
  EC2-C                                                     `10.1.0.0/16` on EC2-C

  EC2-B could not SSH to  TCP/22 timed out                  Added the required SSH
  EC2-C                                                     rule on EC2-C

  EC2-B reached EC2-C but `Permission denied (publickey)`   Reconnected A -\> B
  authentication failed                                     with `ssh -A`

  EC2-C was tested using  Timeout                           Used private IP
  its public IP                                             `10.2.2.173` for the
                                                            TGW test
  ---------------------------------------------------------------------------------

## Key Technical Lessons

### Routing and Security Are Different Layers

A correct route does not automatically mean a protocol is permitted.
Ping success also does not prove SSH is allowed because ICMP and TCP/22
are controlled separately.

### Error Messages Help Identify the Layer

`Connection timed out` generally points toward reachability, routing,
filtering, or port-access issues.

`Permission denied (publickey)` means the SSH service was reached but
authentication failed.

### Use Private IPs for Internal TGW Communication

The internal tests used `10.0.1.29`, `10.1.2.44`, and `10.2.2.173`,
demonstrating private VPC-to-VPC networking.

## Security Considerations

-   Never commit `.pem` or private-key files to GitHub.
-   Avoid opening SSH to `0.0.0.0/0`.
-   Restrict administrative access to trusted sources.
-   Apply least privilege to Security Groups.
-   Use agent forwarding only through trusted hosts.
-   For production, consider AWS Systems Manager Session Manager to
    reduce direct SSH exposure.

Suggested `.gitignore`:

``` gitignore
*.pem
*.key
.env
```

## Skills Demonstrated

-   AWS VPC networking
-   AWS Transit Gateway
-   Hub-and-spoke architecture
-   CIDR planning
-   Subnets and route tables
-   Transit Gateway attachments/routing
-   Amazon EC2
-   Security Groups
-   Linux administration
-   SSH authentication and agent forwarding
-   ICMP and TCP troubleshooting
-   Private vs public IP addressing
-   End-to-end network troubleshooting

## Validation Results

  Test                                  Result
  ------------------------------------- --------
  Workstation -\> EC2-A SSH             PASS
  EC2-A -\> EC2-B ICMP                  PASS
  EC2-A -\> EC2-C ICMP                  PASS
  EC2-A -\> EC2-B SSH                   PASS
  EC2-B -\> EC2-C ICMP                  PASS
  EC2-B -\> EC2-C SSH                   PASS
  Multi-hop SSH agent forwarding        PASS
  Spoke-to-spoke private connectivity   PASS

## Future Improvements

-   Rebuild the architecture with Terraform.
-   Enable VPC Flow Logs.
-   Add CloudWatch monitoring.
-   Use AWS Systems Manager Session Manager.
-   Create separate TGW route tables for stronger segmentation.
-   Add centralized egress or inspection.
-   Automate connectivity validation.
-   Deploy applications in each VPC and test application-layer traffic.

## Conclusion

This project successfully implemented and validated a multi-VPC **AWS
Transit Gateway hub-and-spoke architecture**.

The final environment demonstrated private communication across separate
VPCs, spoke-to-spoke connectivity, ICMP testing, SSH connectivity,
Security Group configuration, and secure multi-hop SSH authentication
using agent forwarding.

Most importantly, the project included realistic troubleshooting and
demonstrated a structured method for isolating routing, firewall,
protocol, addressing, and authentication problems.
