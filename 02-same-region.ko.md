# ② 다른 계정 · 같은 리전 (Frankfurt ↔ Frankfurt)

> 🌐 **Language**: **한국어** | [English](./02-same-region.en.md)
>
> 📚 **가이드 목차**: [① 개요](./01-overview.ko.md) · **② 다른 계정·같은 리전(현재)** · [③ 다른 계정·다른 리전](./03-cross-region.ko.md)

```
계정 A (111111111111)                    계정 B (222222222222)
VPC A (Frankfurt) ◄─── VPC Peering (AWS Backbone) ───► VPC B (Frankfurt)
 10.0.0.0/16                                            10.1.0.0/16
```

**이 시나리오의 특징**
- 두 VPC가 **같은 리전(eu-central-1)**, **다른 계정**
- Security Group에서 **상대 계정의 SG ID를 직접 참조** 가능 (권장)
- `--peer-region` 불필요 · Jumbo Frame 지원

---

## Step 1. Peering Connection 생성 (계정 A · Requester)

**Console**
1. 계정 **A**로 로그인 → Region: **eu-central-1 (Frankfurt)**
2. VPC → **Peering connections** → **Create peering connection**
3. 설정:
   - Name: `a-to-b-frankfurt`
   - VPC ID (Requester): VPC A 선택
   - Account: **Another account** 선택
   - Account ID: `222222222222` (계정 B ID)
   - Region: **This Region (eu-central-1)**
   - VPC ID (Accepter): `vpc-bbbb`
4. **Create** → Status `Pending Acceptance` 확인

**CLI (계정 A 자격증명)**
```bash
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-aaaa \
  --peer-vpc-id vpc-bbbb \
  --peer-owner-id 222222222222 \   # 계정 B ID (필수)
  --region eu-central-1 \
  --profile account-a
# 같은 리전이므로 --peer-region 생략
```

## Step 2. Peering 요청 수락 (계정 B · Accepter)

> ⚠️ 반드시 **계정 B의 자격증명**으로 수락. Requester(계정 A)에서는 수락 불가. 미수락 시 **7일 후 expired**.

```bash
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id pcx-xxxx \
  --region eu-central-1 \
  --profile account-b
```

## Step 3. Route Table 업데이트 (양쪽)

```bash
# 계정 A → B 경로
aws ec2 create-route --route-table-id rtb-aaaa \
  --destination-cidr-block 10.1.0.0/16 \
  --vpc-peering-connection-id pcx-xxxx \
  --region eu-central-1 --profile account-a

# 계정 B → A 경로
aws ec2 create-route --route-table-id rtb-bbbb \
  --destination-cidr-block 10.0.0.0/16 \
  --vpc-peering-connection-id pcx-xxxx \
  --region eu-central-1 --profile account-b
```

## Step 4. Security Group 업데이트 (SG ID 참조 권장)

> **같은 리전 장점** — CIDR 대신 **상대 계정의 SG ID를 직접 참조** 가능. 형식: `account-id/sg-id`

계정 A Security Group (Inbound):

| Type | Protocol | Port | Source |
|------|----------|------|--------|
| Custom TCP | TCP | 필요한 포트 | `222222222222/sg-bbbb` |

계정 B Security Group (Inbound):

| Type | Protocol | Port | Source |
|------|----------|------|--------|
| Custom TCP | TCP | 필요한 포트 | `111111111111/sg-aaaa` |

> SG 참조 대신 CIDR(`10.1.0.0/16`)로 허용해도 됩니다. SG 참조가 IP 변경에 강인하고 관리가 편합니다.

## Step 5. NACL 확인 (커스텀 NACL 사용 시)
양쪽 서브넷 NACL이 상대 VPC CIDR의 **in/out** 및 **return traffic(ephemeral port 1024–65535)**을 허용하는지 확인.

## Step 6. DNS Resolution (선택)
```bash
# 계정 A(Requester) 옵션
aws ec2 modify-vpc-peering-connection-options \
  --vpc-peering-connection-id pcx-xxxx \
  --requester-peering-connection-options '{"AllowDnsResolutionFromRemoteVpc":true}' \
  --region eu-central-1 --profile account-a
# 계정 B(Accepter) 옵션
aws ec2 modify-vpc-peering-connection-options \
  --vpc-peering-connection-id pcx-xxxx \
  --accepter-peering-connection-options '{"AllowDnsResolutionFromRemoteVpc":true}' \
  --region eu-central-1 --profile account-b
```

## Step 7. NAT Gateway 경유 Route 정리
```bash
aws ec2 delete-route --route-table-id rtb-aaaa \
  --destination-cidr-block 10.1.0.0/16 \
  --region eu-central-1 --profile account-a
```

---

## Step 8. 검증 — 양방향 통신 확인 (필수)

> ✅ 아래는 **실측 결과 (2026-07-07)**. team-a(Frankfurt) ↔ team-b(Frankfurt) 같은 리전 크로스 계정 환경에서 실제 EC2로 검증. **SG ID 크로스 계정 참조**까지 확인.
>
> ⚠️ 반드시 **양방향 모두 ping** — 한 방향만 되면 반대쪽 Route/SG 누락.

### ① Peering 상태 확인
```bash
aws ec2 describe-vpc-peering-connections \
  --vpc-peering-connection-ids pcx-xxxx \
  --query 'VpcPeeringConnections[].Status' --region eu-central-1 --profile account-a
```
```json
[ { "Code": "active", "Message": "Active" } ]
```
![Tab2 Peering Status](./images/tab2-1-peering-status.png)

### ② (핵심) Security Group ID 크로스 계정 참조 확인
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
✅ 상대 계정 SG를 `PeeringStatus: active`로 참조 — **다른 리전에서는 불가능한 기능이 같은 리전에서 정상 동작함을 실증**.

![Tab2 SG reference](./images/tab2-2-sg-reference.png)

### ③ A → B 방향 ping (같은 리전, RTT < 1ms)
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
✅ **0% packet loss / 평균 0.75ms**

![Tab2 A to C ping](./images/tab2-3-ping-a-to-c.png)

### ④ B → A 방향 ping (역방향 대칭)
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
✅ **0% packet loss / 평균 0.76ms**

![Tab2 C to A ping](./images/tab2-4-ping-c-to-a.png)

---

## ✅ 판정 기준
- Peering Status = `active`
- SG ID 참조 시 `PeeringStatus: active`
- 양방향 모두 `0% packet loss` (같은 리전 RTT < 1ms)
- traceroute 단일 홉

→ 모두 충족하면 정상. (실측 평균 RTT 0.75ms)

---

### 실측 환경 (참고)
| 구분 | VPC A (Requester) | VPC C (Accepter) |
|------|-------------------|------------------|
| 계정 | 602849569839 (team-a) | 794386800974 (team-b) |
| 리전 | eu-central-1 | eu-central-1 |
| VPC CIDR | 10.0.0.0/16 | 10.2.0.0/16 |
| EC2 Private IP | 10.0.1.77 | 10.2.1.135 |
| Peering ID | pcx-06cb153ca892eaf86 | (`--peer-region` 미사용) |
