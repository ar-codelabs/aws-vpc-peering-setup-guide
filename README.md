# VPC Peering Guide

> 🌐 **Language**: [한국어](#한국어) | [English](#english)

---

## 한국어

AWS VPC Peering 설정 가이드 (시나리오별) + 크로스 계정 실측 검증 결과.

**👉 [① 개요(메인)부터 시작하기](./01-overview.ko.md)** — 개요 문서의 표에서 원하는 시나리오로 이동합니다.

| # | 문서 | 내용 |
|---|------|------|
| ① | **[개요 (메인)](./01-overview.ko.md)** | 개념 · 공통 원리 · 시나리오 비교표 · 시나리오 링크 |
| ② | [다른 계정 · 같은 리전](./02-same-region.ko.md) | Frankfurt ↔ Frankfurt · 설정 + 실측 (SG ID 참조) |
| ③ | [다른 계정 · 다른 리전](./03-cross-region.ko.md) | Frankfurt ↔ Paris · 설정 + 실측 |

✅ **실측 검증 (2026-07-07)**: Tab 2 평균 0.75ms · Tab 3 평균 8.7ms · 양쪽 0% loss · 백본 단일 홉

---

## English

AWS VPC Peering setup guide (per scenario) + real cross-account validation.

**👉 [Start from ① Overview (Main)](./01-overview.en.md)** — jump to your scenario from the table there.

| # | Document | Content |
|---|----------|---------|
| ① | **[Overview (Main)](./01-overview.en.md)** | Concepts · common principles · comparison · scenario links |
| ② | [Different account · Same region](./02-same-region.en.md) | Frankfurt ↔ Frankfurt · setup + measured (SG ID ref) |
| ③ | [Different account · Different region](./03-cross-region.en.md) | Frankfurt ↔ Paris · setup + measured |

✅ **Validated (2026-07-07)**: Tab 2 avg 0.75ms · Tab 3 avg 8.7ms · both 0% loss · single-hop backbone

---

## 📁 구조 / Structure
```
vpc-peering-guide/
├── README.md                  # 진입점 / entry point
├── 01-overview.ko.md          # ① 개요 (메인)
├── 01-overview.en.md          # ① Overview (main)
├── 02-same-region.ko.md       # ② 같은 리전
├── 02-same-region.en.md       # ② Same region
├── 03-cross-region.ko.md      # ③ 다른 리전
├── 03-cross-region.en.md      # ③ Different region
└── images/                    # 캡쳐 이미지 / screenshots
```
