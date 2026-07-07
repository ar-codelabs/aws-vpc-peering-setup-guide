# VPC Peering Guide — Overview (Main)

> 🌐 **Language**: [한국어](./01-overview.ko.md) | **English**
>
> 📚 **Contents**: **① Overview (current)** · [② Different account · Same region](./02-same-region.en.md) · [③ Different account · Different region](./03-cross-region.en.md)

A step-by-step guide to move NAT Gateway-based traffic to VPC Peering (AWS Backbone). Each scenario has its own document.

---

## Choose your scenario

| # | Scenario | Diagram | Guide | Measured |
|---|----------|---------|-------|----------|
| ② | **Different account · Same region** | Frankfurt ↔ Frankfurt | [Open](./02-same-region.en.md) | ✅ avg 0.75ms |
| ③ | **Different account · Different region** | Frankfurt ↔ Paris | [Open](./03-cross-region.en.md) | ✅ avg 8.7ms |

> Same-account peering follows the same principles (just omit the account fields). The no-CIDR-overlap rule always applies.

---

## Why switch to Peering?

```
Before:  VPC A ──→ NAT Gateway ──→ Internet ──→ VPC B
After:   VPC A ◄──── VPC Peering (AWS Backbone) ────► VPC B
```

- **Cost**: ~60–85% NAT Gateway cost reduction
- **Latency**: uses the AWS backbone, no Internet hop
- **Security**: private communication, no Internet exposure

---

## Core principles (common to all scenarios)

| Step | Description |
|------|-------------|
| ① Create | Create the peering request from the Requester VPC (A) |
| ② Accept | Accept from the Accepter VPC (B) side (within 7 days) |
| ③ Routing | Add the peer VPC CIDR route on **both** route tables |
| ④ Firewall | Allow the peer CIDR on **both** Security Groups / NACLs |
| ⑤ (Optional) DNS | Enable private DNS resolution on both sides if needed |

### ⚠️ Three golden rules
1. **Bidirectional required** — configuring Route/SG on only one side breaks connectivity
2. **No CIDR overlap** — overlapping CIDRs fail peering creation (regardless of account/region)
3. **No transitive routing** — A↔B and B↔C do not imply A↔C (each needs its own peering)

---

## Prerequisites checklist

- [ ] Confirm the two VPC CIDR blocks **do not overlap**
- [ ] Confirm both **AWS Account IDs** (12 digits) — required for cross-account
- [ ] Confirm both VPC IDs
- [ ] Confirm both Route Table IDs
- [ ] Check both subnets' NACL policies (if using custom NACLs)
- [ ] List the required ports/protocols
- [ ] IAM: Requester = `ec2:CreateVpcPeeringConnection` / Accepter = `ec2:AcceptVpcPeeringConnection`

---

## Example environment (common notation)

| Field | VPC A | VPC B |
|-------|-------|-------|
| AWS Account ID | 111111111111 | 222222222222 |
| VPC ID | vpc-aaaa | vpc-bbbb |
| CIDR | 10.0.0.0/16 | 10.1.0.0/16 |
| Route Table | rtb-aaaa | rtb-bbbb |

---

## References
- [AWS VPC Peering docs](https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html)
- [Create a peering connection (cross-account/region)](https://docs.aws.amazon.com/vpc/latest/peering/create-vpc-peering-connection.html)
- [Update route tables](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-routing.html)
- [Peering limitations](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-basics.html)
