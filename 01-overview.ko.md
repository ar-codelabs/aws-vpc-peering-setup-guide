# VPC Peering 가이드 — 개요 (메인)

> 🌐 **Language**: **한국어** | [English](./01-overview.en.md)
>
> 📚 **가이드 목차**: **① 개요(현재)** · [② 다른 계정·같은 리전](./02-same-region.ko.md) · [③ 다른 계정·다른 리전](./03-cross-region.ko.md)

NAT Gateway 경유 통신을 VPC Peering(AWS Backbone)으로 전환하는 Step-by-Step 가이드입니다. 시나리오별로 별도 문서를 제공합니다.

---

## 시나리오 선택

아래 표에서 본인 환경에 맞는 가이드로 이동하세요.

| # | 시나리오 | 다이어그램 | 가이드 | 실측 검증 |
|---|----------|-----------|--------|-----------|
| ② | **다른 계정 · 같은 리전** | Frankfurt ↔ Frankfurt | [바로가기](./02-same-region.ko.md) | ✅ 평균 0.75ms |
| ③ | **다른 계정 · 다른 리전** | Frankfurt ↔ Paris | [바로가기](./03-cross-region.ko.md) | ✅ 평균 8.7ms |

> 같은 계정 내 Peering도 원리는 동일합니다(계정 지정만 생략). CIDR 겹침 규칙은 모든 경우에 적용됩니다.

---

## 왜 Peering으로 바꾸나?

```
Before:  VPC A ──→ NAT Gateway ──→ Internet ──→ VPC B
After:   VPC A ◄──── VPC Peering (AWS Backbone) ────► VPC B
```

- **비용 절감**: NAT Gateway 비용 약 60~85% 절감
- **레이턴시 감소**: 인터넷 경유 없이 AWS 백본 사용
- **보안 향상**: 프라이빗 통신, 인터넷 노출 없음

---

## 핵심 원리 (모든 시나리오 공통)

| 단계 | 내용 |
|------|------|
| ① 연결 생성 | Requester VPC(A)에서 Peering 요청 생성 |
| ② 수락 | Accepter VPC(B) 쪽에서 요청 수락 (7일 내) |
| ③ 라우팅 | **양쪽 모두** Route Table에 상대 VPC CIDR 경로 추가 |
| ④ 방화벽 | **양쪽 모두** Security Group / NACL에서 상대 CIDR 허용 |
| ⑤ (선택) DNS | Private DNS resolution 필요 시 양쪽 활성화 |

### ⚠️ Peering의 3대 철칙
1. **양방향 설정 필수** — Route/SG를 한쪽만 하면 통신 안 됨
2. **CIDR 겹치면 불가** — 두 VPC의 CIDR이 겹치면 Peering 생성 실패 (계정/리전 무관)
3. **전이적 라우팅 불가** — A↔B, B↔C가 있어도 A↔C는 자동 연결 안 됨 (각각 별도 Peering 필요)

---

## Prerequisites 체크리스트

- [ ] 두 VPC의 CIDR 블록이 **겹치지 않음** 확인
- [ ] 양쪽 **AWS 계정 ID**(12자리) 확인 — 크로스 계정 시 필수
- [ ] 양쪽 VPC ID 확인
- [ ] 양쪽 Route Table ID 확인
- [ ] 양쪽 서브넷의 NACL 정책 확인 (커스텀 NACL 사용 시)
- [ ] 통신이 필요한 포트/프로토콜 목록 확인
- [ ] IAM 권한: Requester = `ec2:CreateVpcPeeringConnection` / Accepter = `ec2:AcceptVpcPeeringConnection`

---

## 예시 환경 (가이드 공통 표기)

| 항목 | VPC A | VPC B |
|------|-------|-------|
| AWS Account ID | 111111111111 | 222222222222 |
| VPC ID | vpc-aaaa | vpc-bbbb |
| CIDR | 10.0.0.0/16 | 10.1.0.0/16 |
| Route Table | rtb-aaaa | rtb-bbbb |

---

## References
- [AWS VPC Peering 공식 문서](https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html)
- [Peering Connection 생성 (크로스 계정/리전)](https://docs.aws.amazon.com/vpc/latest/peering/create-vpc-peering-connection.html)
- [Route Table 업데이트](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-routing.html)
- [Peering 제약사항](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-basics.html)
