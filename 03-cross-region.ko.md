# ③ 다른 계정 · 다른 리전 (Frankfurt ↔ Paris)

> 🌐 **Language**: **한국어** | [English](./03-cross-region.en.md)
>
> 📚 **가이드 목차**: [① 개요](./01-overview.ko.md) · [② 다른 계정·같은 리전](./02-same-region.ko.md) · **③ 다른 계정·다른 리전(현재)**

```
계정 A (111111111111)                       계정 B (222222222222)
VPC A (Frankfurt / eu-central-1)              VPC B (Paris / eu-west-3)
 10.0.0.0/16  ◄─── VPC Peering (AWS Backbone) ───►  10.1.0.0/16
```

**이 시나리오의 제약 (같은 리전과 다른 점)**
- `--peer-region` **필수**
- Security Group에서 **상대 SG ID 참조 불가** → 반드시 **CIDR**로 허용
- Jumbo Frame 미지원 → **MTU 1500 고정**
- Cross-region **데이터 전송 요금** 발생

---

## Step 1. Peering Connection 생성 (계정 A · Frankfurt · Requester)

**Console**
1. 계정 **A**로 로그인 → Region: **eu-central-1 (Frankfurt)**
2. VPC → **Peering connections** → **Create peering connection**
3. 설정:
   - Name: `frankfurt-to-paris`
   - VPC ID (Requester): VPC A
   - Account: **Another account** → Account ID `222222222222`
   - Region: **Another Region** → `eu-west-3 (Paris)`
   - VPC ID (Accepter): `vpc-bbbb`
4. **Create** → Status `Pending Acceptance`

**CLI (계정 A 자격증명)**
```bash
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-aaaa \
  --peer-vpc-id vpc-bbbb \
  --peer-owner-id 222222222222 \   # 계정 B ID (필수)
  --peer-region eu-west-3 \        # 다른 리전 (필수)
  --region eu-central-1 \
  --profile account-a
```

## Step 2. Peering 요청 수락 (계정 B · Paris · Accepter)

> ⚠️ 반드시 **계정 B의 자격증명**으로, **Accepter 리전(eu-west-3)**에서 수락. 미수락 시 **7일 후 expired**.

```bash
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id pcx-xxxx \
  --region eu-west-3 \             # Accepter 리전
  --profile account-b
```

## Step 3. Route Table 업데이트 (양쪽 · 각자 리전)

```bash
# Frankfurt → Paris (계정 A / eu-central-1)
aws ec2 create-route --route-table-id rtb-aaaa \
  --destination-cidr-block 10.1.0.0/16 \
  --vpc-peering-connection-id pcx-xxxx \
  --region eu-central-1 --profile account-a

# Paris → Frankfurt (계정 B / eu-west-3)
aws ec2 create-route --route-table-id rtb-bbbb \
  --destination-cidr-block 10.0.0.0/16 \
  --vpc-peering-connection-id pcx-xxxx \
  --region eu-west-3 --profile account-b
```

## Step 4. Security Group 업데이트 (CIDR 필수)

> **Cross-region 제약** — 상대 VPC의 SG ID 참조 **불가**. 반드시 **CIDR**로 지정.

Frankfurt Security Group (Inbound):

| Type | Protocol | Port | Source | Description |
|------|----------|------|--------|-------------|
| Custom TCP | TCP | 필요한 포트 | `10.1.0.0/16` | Paris VPC inbound |

Paris Security Group (Inbound):

| Type | Protocol | Port | Source | Description |
|------|----------|------|--------|-------------|
| Custom TCP | TCP | 필요한 포트 | `10.0.0.0/16` | Frankfurt VPC inbound |

## Step 5. NACL 확인 (커스텀 NACL 사용 시)
양쪽 서브넷 NACL이 상대 VPC CIDR의 **in/out** 및 **return traffic(ephemeral port 1024–65535)**을 허용하는지 확인.

## Step 6. DNS Resolution (선택)
```bash
# Frankfurt(Requester) — 계정 A
aws ec2 modify-vpc-peering-connection-options \
  --vpc-peering-connection-id pcx-xxxx \
  --requester-peering-connection-options '{"AllowDnsResolutionFromRemoteVpc":true}' \
  --region eu-central-1 --profile account-a
# Paris(Accepter) — 계정 B
aws ec2 modify-vpc-peering-connection-options \
  --vpc-peering-connection-id pcx-xxxx \
  --accepter-peering-connection-options '{"AllowDnsResolutionFromRemoteVpc":true}' \
  --region eu-west-3 --profile account-b
```

## Step 7. MTU 확인 · NAT 정리
> **MTU** — Cross-region은 **MTU 1500** 고정. 인스턴스 MTU가 9001(jumbo)이면 간헐적 timeout 가능 → 1500으로 조정.
```bash
aws ec2 delete-route --route-table-id rtb-aaaa \
  --destination-cidr-block 10.1.0.0/16 \
  --region eu-central-1 --profile account-a
```

---

## Step 8. 검증 — 양방향 통신 확인 (필수)

> ✅ 아래는 **실측 결과 (2026-07-07)**. team-a(Frankfurt) ↔ team-b(Paris) 크로스 계정·크로스 리전 환경에서 실제 EC2로 검증.
>
> ⚠️ 반드시 **양방향 모두 ping**.

### ① Peering 상태 확인
```bash
aws ec2 describe-vpc-peering-connections \
  --vpc-peering-connection-ids pcx-xxxx \
  --query 'VpcPeeringConnections[].Status' --region eu-central-1 --profile account-a
```
```json
[ { "Code": "active", "Message": "Active" } ]
```
![Tab3 Peering Status](./images/tab3-1-peering-status.png)

### ② A → B 방향 ping (Frankfurt → Paris)
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
✅ **0% packet loss / 평균 8.74ms**

![Tab3 A to B ping](./images/tab3-2-ping-a-to-b.png)

### ③ B → A 방향 ping (Paris → Frankfurt, 역방향 대칭)
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
✅ **0% packet loss / 평균 8.58ms**

![Tab3 B to A ping](./images/tab3-3-ping-b-to-a.png)

### ④ traceroute — AWS 백본 직결(단일 홉) 확인
```bash
traceroute -n 10.1.1.37
```
```
traceroute to 10.1.1.37 (10.1.1.37), 30 hops max, 60 byte packets
 1  10.1.1.37  8.511 ms
```
✅ **단일 홉** — NAT/인터넷 미경유, AWS 백본 직결. 중간에 NAT/public IP 홉이 보이면 Route Table의 more specific route(상대 CIDR → pcx) 확인.

![Tab3 traceroute](./images/tab3-4-traceroute.png)

---

## ✅ 판정 기준
- Peering Status = `active`
- A→B, B→A **양방향 모두** `0% packet loss`
- traceroute 단일 홉 (백본 직결, TTL 127)

→ 세 가지 모두 충족하면 정상. (실측 평균 RTT 8.7ms)

---

## 비용 비교 (Cross-region 예시)
| 항목 | NAT Gateway 경유 | VPC Peering |
|------|------------------|-------------|
| 시간당 고정비 | ~$0.045/h × 2 (양쪽) | 무료 |
| 데이터 처리 | $0.045/GB (NAT) + 인터넷 전송비 | $0.02/GB (cross-region 전송만) |
| 월 1TB 전송 기준 | ~$155 | ~$20 |
| 예상 절감 | - | **약 85% 절감** |
