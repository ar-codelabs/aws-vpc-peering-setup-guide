# ③ Different Account · Different Region (Frankfurt ↔ Paris)

> 🌐 **Language**: [한국어](./03-cross-region.ko.md) | **English**
>
> 📚 **Contents**: [① Overview](./01-overview.en.md) · [② Different account · Same region](./02-same-region.en.md) · **③ Different account · Different region (current)**

```
Account A (111111111111)                    Account B (222222222222)
VPC A (Frankfurt / eu-central-1)             VPC B (Paris / eu-west-3)
 10.0.0.0/16  ◄─── VPC Peering (AWS Backbone) ───►  10.1.0.0/16
```

**Scenario constraints (vs same region)**
- `--peer-region` **required**
- Security Group **cannot reference peer SG ID** → must use **CIDR**
- Jumbo Frame not supported → **MTU 1500 fixed**
- Cross-region **data transfer cost** applies

---

## Step 1. Create Peering Connection (Account A · Frankfurt · Requester)

**Console**
1. Sign in to Account **A** → Region: **eu-central-1 (Frankfurt)**
2. VPC → **Peering connections** → **Create peering connection**
3. Settings:
   - Name: `frankfurt-to-paris`
   - VPC ID (Requester): VPC A
   - Account: **Another account** → Account ID `222222222222`
   - Region: **Another Region** → `eu-west-3 (Paris)`
   - VPC ID (Accepter): `vpc-bbbb`
4. **Create** → Status `Pending Acceptance`

**CLI (Account A credentials)**
```bash
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-aaaa \
  --peer-vpc-id vpc-bbbb \
  --peer-owner-id 222222222222 \   # Account B ID (required)
  --peer-region eu-west-3 \        # Different region (required)
  --region eu-central-1 \
  --profile account-a
```

## Step 2. Accept the Request (Account B · Paris · Accepter)

> ⚠️ Must accept with **Account B credentials** in the **Accepter region (eu-west-3)**. Expires after **7 days** if not accepted.

```bash
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id pcx-xxxx \
  --region eu-west-3 \             # Accepter region
  --profile account-b
```

## Step 3. Update Route Tables (both sides, each region)

```bash
# Frankfurt → Paris (Account A / eu-central-1)
aws ec2 create-route --route-table-id rtb-aaaa \
  --destination-cidr-block 10.1.0.0/16 \
  --vpc-peering-connection-id pcx-xxxx \
  --region eu-central-1 --profile account-a

# Paris → Frankfurt (Account B / eu-west-3)
aws ec2 create-route --route-table-id rtb-bbbb \
  --destination-cidr-block 10.0.0.0/16 \
  --vpc-peering-connection-id pcx-xxxx \
  --region eu-west-3 --profile account-b
```

## Step 4. Update Security Groups (CIDR required)

> **Cross-region constraint** — peer SG ID reference is **not allowed**. Use **CIDR**.

Frankfurt Security Group (Inbound):

| Type | Protocol | Port | Source | Description |
|------|----------|------|--------|-------------|
| Custom TCP | TCP | required port | `10.1.0.0/16` | Paris VPC inbound |

Paris Security Group (Inbound):

| Type | Protocol | Port | Source | Description |
|------|----------|------|--------|-------------|
| Custom TCP | TCP | required port | `10.0.0.0/16` | Frankfurt VPC inbound |

## Step 5. Check NACLs (if using custom NACLs)
Ensure both subnets' NACLs allow the peer VPC CIDR **in/out** and **return traffic (ephemeral ports 1024–65535)**.

## Step 6. DNS Resolution (optional)
```bash
# Frankfurt (Requester) — Account A
aws ec2 modify-vpc-peering-connection-options \
  --vpc-peering-connection-id pcx-xxxx \
  --requester-peering-connection-options '{"AllowDnsResolutionFromRemoteVpc":true}' \
  --region eu-central-1 --profile account-a
# Paris (Accepter) — Account B
aws ec2 modify-vpc-peering-connection-options \
  --vpc-peering-connection-id pcx-xxxx \
  --accepter-peering-connection-options '{"AllowDnsResolutionFromRemoteVpc":true}' \
  --region eu-west-3 --profile account-b
```

## Step 7. MTU check · NAT cleanup
> **MTU** — Cross-region is **MTU 1500** fixed. If instance MTU is 9001 (jumbo), intermittent timeouts may occur → set to 1500.
```bash
aws ec2 delete-route --route-table-id rtb-aaaa \
  --destination-cidr-block 10.1.0.0/16 \
  --region eu-central-1 --profile account-a
```

---

## Step 8. Validation — bidirectional connectivity (required)

> ✅ Below are **measured results (2026-07-07)** from real EC2s in a cross-account cross-region setup (team-a Frankfurt ↔ team-b Paris).
>
> ⚠️ Always ping **both directions**.

### ① Peering status
```bash
aws ec2 describe-vpc-peering-connections \
  --vpc-peering-connection-ids pcx-xxxx \
  --query 'VpcPeeringConnections[].Status' --region eu-central-1 --profile account-a
```
```json
[ { "Code": "active", "Message": "Active" } ]
```
![Tab3 Peering Status](./images/tab3-1-peering-status.png)

### ② A → B ping (Frankfurt → Paris)
```
PING 10.1.1.37 (10.1.1.37) 56(84) bytes of data.
64 bytes from 10.1.1.37: icmp_seq=1 ttl=127 time=8.87 ms
64 bytes from 10.1.1.37: icmp_seq=2 ttl=127 time=8.70 ms
64 bytes from 10.1.1.37: icmp_seq=3 ttl=127 time=8.70 ms
64 bytes from 10.1.1.37: icmp_seq=4 ttl=127 time=8.68 ms
--- 10.1.1.37 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3006ms
rtt min/avg/max/mdev = 8.675/8.736/8.871/0.078 ms
```
✅ **0% packet loss / avg 8.74ms**

![Tab3 A to B ping](./images/tab3-2-ping-a-to-b.png)

### ③ B → A ping (Paris → Frankfurt, reverse symmetric)
```
PING 10.0.1.77 (10.0.1.77) 56(84) bytes of data.
64 bytes from 10.0.1.77: icmp_seq=1 ttl=127 time=8.58 ms
64 bytes from 10.0.1.77: icmp_seq=2 ttl=127 time=8.58 ms
64 bytes from 10.0.1.77: icmp_seq=3 ttl=127 time=8.58 ms
64 bytes from 10.0.1.77: icmp_seq=4 ttl=127 time=8.58 ms
--- 10.0.1.77 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 8.578/8.579/8.582/0.001 ms
```
✅ **0% packet loss / avg 8.58ms**

![Tab3 B to A ping](./images/tab3-3-ping-b-to-a.png)

### ④ traceroute — verify single-hop backbone
```bash
traceroute -n 10.1.1.37
```
```
traceroute to 10.1.1.37 (10.1.1.37), 30 hops max, 60 byte packets
 1  10.1.1.37  8.511 ms
```
✅ **Single hop** — no NAT/Internet; direct over the AWS backbone. If a NAT/public-IP hop appears, check the more specific route (peer CIDR → pcx) in the route table.

![Tab3 traceroute](./images/tab3-4-traceroute.png)

---

## ✅ Pass criteria
- Peering Status = `active`
- Both A→B and B→A `0% packet loss`
- traceroute single hop (backbone direct, TTL 127)

→ All met = healthy. (Measured avg RTT 8.7ms)

---

## Cost comparison (cross-region example)
| Item | via NAT Gateway | VPC Peering |
|------|-----------------|-------------|
| Hourly fixed | ~$0.045/h × 2 (both sides) | Free |
| Data processing | $0.045/GB (NAT) + Internet egress | $0.02/GB (cross-region transfer only) |
| Per 1TB/month | ~$155 | ~$20 |
| Estimated savings | - | **~85%** |
