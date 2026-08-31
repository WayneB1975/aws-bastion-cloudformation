# Multi-AZ VPC with Bastion Host — CloudFormation

A single CloudFormation template that provisions an isolated multi-AZ VPC with a bastion host and two private application instances. Built twice — once through the console, once as code — as a study in what infrastructure-as-code actually buys you.

> Full write-up: [I Built the Same Bastion Architecture Twice](http://LINK)

---

## Architecture

                        Internet

                            │

                    ┌───────▼───────┐

                    │ Internet GW   │

                    └───────┬───────┘

                            │

  ┌─────────────────────────┼─────────────────────────┐

  │  VPC 172.16.0.0/16      │                         │

  │                         │                         │

  │   AZ-a                  │            AZ-b         │

  │   ┌──────────────────┐  │  ┌──────────────────┐   │

  │   │ Public 1A        │  │  │ Public 2B        │   │

  │   │ 172.16.1.0/24    │◄─┴─►│ 172.16.4.0/24    │   │

  │   │                  │     │  ┌────────────┐  │   │

  │   │                  │     │  │  Bastion   │  │   │

  │   └──────────────────┘     │  └─────┬──────┘  │   │

  │                            └────────┼─────────┘   │

  │   ┌──────────────────┐        SSH   │             │

  │   │ App Private 1A   │◄─────────────┘             │

  │   │ 172.16.2.0/24    │                            │

  │   │  ┌────────────┐  │      ┌──────────────────┐  │

  │   │  │    App1    │──┼─────►│ App Private 2B   │  │

  │   │  └────────────┘  │ ICMP │ 172.16.5.0/24    │  │

  │   └──────────────────┘      │  ┌────────────┐  │  │

  │                             │  │    App2    │  │  │

  │   ┌──────────────────┐      │  └────────────┘  │  │

  │   │ Data Private 1A  │      └──────────────────┘  │

  │   │ 172.16.3.0/24    │      ┌──────────────────┐  │

  │   └──────────────────┘      │ Data Private 2B  │  │

  │                             │ 172.16.6.0/24    │  │

  │                             └──────────────────┘  │

  └───────────────────────────────────────────────────┘

**Resources created:** 1 VPC, 6 subnets, 1 internet gateway, 1 public route table with 2 associations, 3 security groups, 3 EC2 instances.

---

## Design decisions

| Decision | Rationale |
| :---- | :---- |
| **No NAT Gateway** | Application instances must have no internet path, inbound or outbound. All required traffic is east-west inside the VPC and never needs internet infrastructure. |
| **Private subnets use the main route table** | Local-only routing. No explicit association needed to achieve isolation. |
| **Security groups reference other security groups, not CIDRs** | `SourceSecurityGroupId` survives instance replacement and scaling. A CIDR hardcodes an assumption about addressing. |
| **App2 has no key pair and no SSH ingress** | Its only function is to answer a ping and prove cross-AZ east-west connectivity. Least privilege: a shell it never needs is a shell that can't be abused. |
| **Data subnets provisioned but unused** | Reserved for a future database tier. Costs nothing to define. |
| **AMI resolved via SSM parameter** | Region-portable and always current. Hardcoded AMI IDs pin a template to one region and one point in time. |
| **Source IP is a required parameter** | No home IP addresses in version control. |

---

## Prerequisites

- An AWS account and the AWS CLI configured (`aws configure`)  
- An existing EC2 key pair in your target region  
- Your current public IP — get it from [whatismyipaddress.com](https://whatismyipaddress.com)

---

## Deploy

aws cloudformation deploy \\

  \--template-file bastion-vpc-template.yaml \\

  \--stack-name vpc-stack \\

  \--parameter-overrides \\

      MyIpCidr=YOUR.PUBLIC.IP.HERE/32 \\

      KeyPairName=YourKeyPairName

Retrieve the connection details:

aws cloudformation describe-stacks \\

  \--stack-name vpc-stack \\

  \--query 'Stacks\[0\].Outputs' \\

  \--output table

### Parameters

| Parameter | Default | Description |
| :---- | :---- | :---- |
| `MyIpCidr` | *required* | Your public IP in CIDR form. Validated by regex. |
| `KeyPairName` | `Bastion` | An existing EC2 key pair name. |
| `InstanceType` | `t2.micro` | One of `t2.micro`, `t3.micro`, `t3.small`. |
| `LatestAmiId` | SSM path | Resolves to current Amazon Linux 2023\. Leave as-is. |

---

## Verify

**1\. Connect to the bastion.** The stack outputs a ready-to-paste command.

ssh \-i Bastion.pem ec2-user@\<BastionPublicIp\>

**2\. Reach App1 through the bastion.** Use ProxyJump — the private key stays on your workstation and never touches the internet-facing host.

ssh \-i Bastion.pem \-J ec2-user@\<BastionPublicIp\> ec2-user@\<App1PrivateIp\>

**3\. From App1, ping App2.**

ping \<App2PrivateIp\>

Expect replies. This confirms east-west connectivity between private subnets across availability zones.

### Expected failures

These are the design working correctly, not bugs:

| Test | Result | Why |
| :---- | :---- | :---- |
| Ping App2 from the **bastion** | 100% packet loss | App2 permits ICMP only from App1's security group. |
| SSH to App2 from anywhere | Refused | No key pair, no port 22 ingress. By design. |
| `curl google.com` from App1 or App2 | Timeout | No NAT Gateway. Private instances have no outbound internet path. |

---

## Connecting cleanly

Add to `~/.ssh/config` to reduce the whole thing to `ssh app1`:

Host bastion

    HostName \<BastionPublicIp\>

    User ec2-user

    IdentityFile \~/.ssh/Bastion.pem

Host app1

    HostName \<App1PrivateIp\>

    User ec2-user

    IdentityFile \~/.ssh/Bastion.pem

    ProxyJump bastion

**Do not copy the .pem file onto the bastion.** The bastion is the one internet-facing host in this architecture; a private key stored there gives anyone who compromises it access to everything behind it. ProxyJump achieves the same result with the key never leaving your machine.

---

## Teardown

Delete the stack when you're finished. Everything provisioned by this template is removed in one operation — no orphaned subnets, no forgotten security groups, no instance still running in a region you forgot you used.

aws cloudformation delete-stack \--stack-name vpc-stack

The call returns immediately; deletion runs asynchronously. Wait for it to finish:

aws cloudformation wait stack-delete-complete \--stack-name vpc-stack

### Verify it actually deleted

Do not assume. `delete-stack` can fail partway and leave resources behind, and a stack in `DELETE_FAILED` still bills you for whatever survived.

\# Should return an error saying the stack does not exist

aws cloudformation describe-stacks \--stack-name vpc-stack

\# Should return no instances

aws ec2 describe-instances \\

  \--filters "Name=tag:Name,Values=BastionHost,App1,App2" \\

            "Name=instance-state-name,Values=running,stopped" \\

  \--query 'Reservations\[\].Instances\[\].\[InstanceId,State.Name\]' \\

  \--output table

\# Should return only the default VPC

aws ec2 describe-vpcs \--query 'Vpcs\[\].\[VpcId,CidrBlock,IsDefault\]' \--output table

If the third command still shows `172.16.0.0/16`, the stack did not fully delete.

**Check other regions too.** The console only shows one region at a time, and an instance running in a region you're not looking at bills exactly the same as one you can see.

for region in us-east-1 us-east-2 us-west-1 us-west-2; do

  echo "== $region"

  aws ec2 describe-instances \--region $region \\

    \--filters "Name=instance-state-name,Values=running" \\

    \--query 'Reservations\[\].Instances\[\].InstanceId' \--output text

done

### If deletion fails

The usual cause is a dependency created outside the stack. A manually-added security group rule, an ENI, or a manually-launched instance in one of the stack's subnets will block the VPC from deleting.

Check the stack's **Events** tab in the console — CloudFormation names the specific resource it couldn't delete and why. Remove that resource manually, then rerun `delete-stack`.

### Standing guardrail

Independent of any single stack: set a billing alarm so idle resources surface on their own rather than on your statement.

Billing console → **Budgets** → Create budget → Cost budget → set a low monthly threshold with an email alert. Ten minutes once, and it catches the resource you forget rather than the one you remember.

---

## Cost

All three instances are t2.micro and eligible for the AWS Free Tier. There is no NAT Gateway, which is the component that usually generates unexpected charges in VPC labs. Outside the Free Tier, expect a few cents per hour.

Delete the stack when you're done.

---

## Troubleshooting

SSH reports three distinct failures. Each points at a different layer:

| Error | Layer | Check |
| :---- | :---- | :---- |
| `Connection timed out` | Network | Security groups, NACLs, route tables, subnet placement, public IP |
| `Connection refused` | Service | sshd not running, or wrong port |
| `Permission denied (publickey)` | Credential | Wrong key, wrong username, or key file permissions |

Three specific cases worth naming:

**Timeout on a bastion that worked yesterday** — your public IP almost certainly changed. Residential ISPs rotate addresses on DHCP lease renewal, router reboots, and maintenance windows. The `/32` rule in the security group now points at someone else's address.

MYIP=$(curl \-s https://checkip.amazonaws.com)

aws ec2 authorize-security-group-ingress \\

  \--group-id \<BastionSgId\> \--protocol tcp \--port 22 \--cidr ${MYIP}/32

Revoke the stale rule afterward with `revoke-security-group-ingress`, or the group accumulates addresses that no longer belong to you.

**`Permissions 0664 for 'x.pem' are too open`** — SSH ignores the key entirely, then reports `Permission denied (publickey)` several lines later. The real cause is higher in the output. Fix with `chmod 400 x.pem`. Using ProxyJump avoids this class of problem altogether, since the key never leaves your workstation.

**`SSH protocol v1 is no longer supported`** — you typed `-1` (the digit) instead of `-i` (the letter). They're nearly identical in most terminal fonts.

For anything else, VPC → **Reachability Analyzer** will identify the exact hop that's dropping traffic.

---

## Repository contents

.

├── bastion-vpc-template.yaml    \# The CloudFormation template

├── README.md

└── docs/

    └── architecture.png         \# Diagram

---

## License

MIT  
