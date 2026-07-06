# GCP 인프라 구성 상세 (Console 기준)

이 프로젝트의 GCP 측 인프라는 Terraform이 아닌 **GCP 콘솔에서 직접 구성**했습니다.
아래는 실제로 설정한 값을 그대로 정리한 문서입니다.

---

## 1. VPC 네트워크

| 항목 | 값 |
|---|---|
| VPC 이름 | `history-net` |
| 서브넷 생성 모드 | Custom (자동 생성 안 함) |
| 라우팅 모드 | Regional |

## 2. 서브넷

| 항목 | 값 |
|---|---|
| 서브넷 이름 | `history-net-seoul` |
| 리전 | `asia-northeast3` (서울) |
| IP 범위 | `10.200.1.0/24` |
| Private Google Access | 사용 |

## 3. 방화벽 규칙

| 이름 | 방향 | 프로토콜/포트 | 소스 범위 | 대상 태그 | 용도 |
|---|---|---|---|---|---|
| `allow-ssh-bastion` | 인그레스 | TCP 22 | 관리자 접속 IP | `bastion` | Bastion SSH 접속 허용 |
| `allow-internal` | 인그레스 | TCP/ICMP 전체 | `10.200.1.0/24` | 전체 | 서브넷 내부 통신 허용 (Bastion → Cloud SQL 등) |

## 4. Compute Engine — Bastion 서버

| 항목 | 값 |
|---|---|
| 인스턴스 이름 | `gcp-bastion` |
| 이미지 | Rocky Linux 9 |
| 머신 유형 | e2-small |
| 네트워크 태그 | `bastion` |
| 외부 IP | 임시 외부 IP 할당 (SSH 접속용) |
| 네트워크 | `history-net` / `history-net-seoul` |

**SSH 접속 확인**: 외부에서 Bastion으로 SSH 접속 성공 (`screenshots/02-bastion-ssh-connected.png` 참고)

## 5. Cloud SQL (MySQL) — Private Service Access

| 항목 | 값 |
|---|---|
| 데이터베이스 엔진 | MySQL 8.0 |
| 연결 방식 | Private IP (Public IP 미사용) |
| 연결 대상 VPC | `history-net` |

**PSA(Private Service Access) 구성 순서**
1. VPC 네트워크 > 프라이빗 서비스 연결에서 IP 주소 범위 할당 (자동 할당, `/16`)
2. `servicenetworking.googleapis.com`과 VPC 피어링 연결 생성
3. Cloud SQL 인스턴스 생성 시 "프라이빗 IP" 옵션으로 위에서 만든 VPC(`history-net`) 선택
4. 인스턴스 생성 후 발급된 Private IP 확인

**검증 결과**
- Cloud SQL 데이터베이스 생성 완료 (`screenshots/03-cloudsql-database-created.png`)
- 테이블 생성 쿼리 실행 성공 (`screenshots/04-cloudsql-table-created.png`)
- AWS Bastion에서 `mysql -h db.cloud.local` 명령으로 GCP Cloud SQL 접속 성공 (`screenshots/05-aws-to-cloudsql-verified.png`)

## 6. Route53 사설 도메인 연결

- Route53에 `db.cloud.local` 레코드를 Cloud SQL의 Private IP를 가리키는 A 레코드로 등록
- AWS 측 Bastion에서 해당 도메인으로 Cloud SQL Private IP까지 정상 접속되는 것을 확인

---

## 참고 — 담당 범위 안내

VPN·BGP(Cloud Router, HA VPN Gateway, VPN 터널 구성)는 팀원이 담당한 영역이라 이 문서에는 포함하지 않았습니다. 본인 담당 범위는 위 1~6번(VPC·서브넷·방화벽·Bastion·Cloud SQL PSA·Route53 연결)입니다.
