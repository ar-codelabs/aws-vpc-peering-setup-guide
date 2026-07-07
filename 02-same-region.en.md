# ② Different Account · Same Region (Frankfurt ↔ Frankfurt)

> 🌐 **Language**: [한국어](./02-same-region.ko.md) | **English**
>
> 📚 **Contents**: [① Overview](./01-overview.en.md) · **② Different account · Same region (current)** · [③ Different account · Different region](./03-cross-region.en.md)

```
Account A (111111111111)                 Account B (222222222222)
VPC A (Frankfurt) ◄─── VPC Peering (AWS Backbone) ───► VPC B (Frankfurt)
 10.0.0.0/16                                            10.1.0.0/16
```

**Scenario characteristics**
- Both VPCs in the **same region (eu-central-1)**, **different accounts**
- Security Group can **reference the peer account's SG ID directly** (recommended)
- No `--peer-region` needed · Jumbo Frame supported

---

## Step 1. Create Peering Connection (Account A · Requester)

**Console**
1. Sign in to Account **A** → Region: **eu-central-1 (Frankfurt)**
2. VPC → **Peering connections** → **Create peering connection**
3. Settings:
   - Name: `a-to-b-frankfurt`
   - VPC ID (Requester): VPC A
   - Account: **Another account**
   - Account ID: `222222222222` (Account B)
   - Region: **This Region (eu-central-1)**
   - VPC ID (Accepter): `vpc-bbbb`
4. **Create** → Status `Pending Acceptance`

**CLI (Account A credentials)**
```bash
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-aaaa \
  --peer-vpc-id vpc-bbbb \
  --peer-owner-id 222222222222 \   # Account B ID (required)
  --region eu-central-1 \
  --profile account-a
# --peer-region omitted (same region)
```

## Step 2. Accept the Request (Account B · Accepter)

> ⚠️ Must accept with **Account B credentials**. The Requester (Account A) cannot accept. Expires after **7 days** if not accepted.

```bash
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id pcx-xxxx \
  --region eu-central-1 \
  --profile account-b
```

## Step 3. Update Route Tables (both sides)

```bash
# Account A → B
aws ec2 create-route --route-table-id rtb-aaaa \
  --destination-cidr-block 10.1.0.0/16 \
  --vpc-peering-connection-id pcx-xxxx \
  --region eu-central-1 --profile account-a

# Account B → A
aws ec2 create-route --route-table-id rtb-bbbb \
  --destination-cidr-block 10.0.0.0/16 \
  --vpc-peering-connection-id pcx-xxxx \
  --region eu-central-1 --profile account-b
```

## Step 4. Update Security Groups (SG ID reference recommended)

> **Same-region advantage** — reference the peer account's SG ID instead of CIDR. Format: `account-id/sg-id`

Account A Security Group (Inbound):

| Type | Protocol | Port | Source |
|------|----------|------|--------|
| Custom TCP | TCP | required port | `222222222222/sg-bbbb` |

Account B Security Group (Inbound):

| Type | Protocol | Port | Source |
|------|----------|------|--------|
| Custom TCP | TCP | required port | `111111111111/sg-aaaa` |

> You may use CIDR (`10.1.0.0/16`) instead. SG reference is more robust to IP changes and easier to manage.

## Step 5. Check NACLs (if using custom NACLs)
Ensure both subnets' NACLs allow the peer VPC CIDR **in/out** and **return traffic (ephemeral ports 1024–65535)**.

## Step 6. DNS Resolution (optional)
```bash
# Account A (Requester)
aws ec2 modify-vpc-peering-connection-options \
  --vpc-peering-connection-id pcx-xxxx \
  --requester-peering-connection-options '{"AllowDnsResolutionFromRemoteVpc":true}' \
  --region eu-central-1 --profile account-a
# Account B (Accepter)
aws ec2 modify-vpc-peering-connection-options \
  --vpc-peering-connection-id pcx-xxxx \
  --accepter-peering-connection-options '{"AllowDnsResolutionFromRemoteVpc":true}' \
  --region eu-central-1 --profile account-b
```

## Step 7. Clean up NAT Gateway routes
```bash
aws ec2 delete-route --route-table-id rtb-aaaa \
  --destination-cidr-block 10.1.0.0/16 \
  --region eu-central-1 --profile account-a
```

---

## Step 8. Validation — bidirectional connectivity (required)

> ✅ Below are **measured results (2026-07-07)** from real EC2s in a same-region cross-account setup (team-a Frankfurt ↔ team-b Frankfurt), including **cross-account SG ID reference**.
>
> ⚠️ Always ping **both directions** — if only one works, the other side's Route/SG is missing.

### ① Peering status
```bash
aws ec2 describe-vpc-peering-connections \
  --vpc-peering-connection-ids pcx-xxxx \
  --query 'VpcPeeringConnections[].Status' --region eu-central-1 --profile account-a
```
```json
[ { "Code": "active", "Message": "Active" } ]
```
![Tab2 Peering Status](./images/tab2-1-peering-status.png)

### ② (Key) Verify cross-account SG ID reference
```bash
aws ec2 describe-security-groups --group-ids sg-aaaa \
  --query 'SecurityGroups[0].IpPermissions[].UserIdGroupPairs[]' \
  --region eu-central-1 --profile account-a
```
```json
[
    {
        "Description": "peer-sg-c-ref",
        "UserId": "222222222222",
        "GroupId": "sg-bbbb",
        "VpcPeeringConnectionId": "pcx-xxxx",
        "PeeringStatus": "active"
    }
]
```
✅ References the peer account's SG with `PeeringStatus: active` — a feature unavailable cross-region, proven working in the same region.

![Tab2 SG reference](./images/tab2-2-sg-reference.png)

### ③ A → B ping (same region, RTT < 1ms)
```
PING 10.2.1.135 (10.2.1.135) 56(84) bytes of data.
64 bytes from 10.2.1.135: icmp_seq=1 ttl=127 time=0.707 ms
64 bytes from 10.2.1.135: icmp_seq=2 ttl=127 time=0.833 ms
64 bytes from 10.2.1.135: icmp_seq=3 ttl=127 time=0.714 ms
64 bytes from 10.2.1.135: icmp_seq=4 ttl=127 time=0.733 ms
--- 10.2.1.135 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3093ms
rtt min/avg/max/mdev = 0.707/0.746/0.833/0.050 ms
```
✅ **0% packet loss / avg 0.75ms**

![Tab2 A to C ping](./images/tab2-3-ping-a-to-c.png)

### ④ B → A ping (reverse, symmetric)
```
PING 10.0.1.77 (10.0.1.77) 56(84) bytes of data.
64 bytes from 10.0.1.77: icmp_seq=1 ttl=127 time=0.703 ms
64 bytes from 10.0.1.77: icmp_seq=2 ttl=127 time=0.819 ms
64 bytes from 10.0.1.77: icmp_seq=3 ttl=127 time=0.707 ms
64 bytes from 10.0.1.77: icmp_seq=4 ttl=127 time=0.800 ms
--- 10.0.1.77 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3028ms
rtt min/avg/max/mdev = 0.703/0.757/0.819/0.052 ms
```
✅ **0% packet loss / avg 0.76ms**

![Tab2 C to A ping](./images/tab2-4-ping-c-to-a.png)

---

## ✅ Pass criteria
- Peering Status = `active`
- SG ID reference shows `PeeringStatus: active`
- Both directions `0% packet loss` (same-region RTT < 1ms)
- traceroute single hop

→ All met = healthy. (Measured avg RTT 0.75ms)

---

### Measured environment (reference)
| Field | VPC A (Requester) | VPC C (Accepter) |
|-------|-------------------|------------------|
| Account | 602849569839 (team-a) | 794386800974 (team-b) |
| Region | eu-central-1 | eu-central-1 |
| VPC CIDR | 10.0.0.0/16 | 10.2.0.0/16 |
| EC2 Private IP | 10.0.1.77 | 10.2.1.135 |
| Peering ID | pcx-06cb153ca892eaf86 | (no `--peer-region`) |
