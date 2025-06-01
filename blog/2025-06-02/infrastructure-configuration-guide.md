---
slug: infrastructure-configuration-guide
title: AWS 기반 원큐 오더 인프라 구성기
authors: polar
tags: []
---

# AWS 기반 원큐 오더 인프라 구성기: 비용 효율적 아키텍처 설계

안녕하세요, 원큐 오더 PL 남승현입니다.

이전 포스팅에서 [AWS EKS 인스턴스 스펙을 결정한 이유](../2025-06-01/aws-eks-instance-spec-decision.md)에 대해 말씀드렸는데요,
오늘은 실제로 AWS 환경에서 어떻게 인프라를 구성했는지 상세히 공유해보려고 해요.

## 인프라 구성 요구사항

저희 원큐 오더 프로젝트는 금융 서비스 특성상 다음과 같은 요구사항이 있었어요.

- **보안성**: 금융 데이터 처리를 위한 강화된 보안 정책
- **가용성**: 서비스 중단 최소화를 위한 고가용성 설계
- **확장성**: 트래픽 증가에 대응할 수 있는 확장 가능한 아키텍처

하지만 현실적인 제약사항도 고려해야 했어요.

- **예산 제한**: 총 25만원 크레딧으로 프로젝트 기간(4월 18일부터 6월 11일) 동안 운영
- **팀 규모**: 1명이 인프라를 관리

## 아키텍처 개요

<!-- ![AWS 인프라 아키텍처](./img/wonq-order-aws-architecture.png) -->

최종적으로 결정한 인프라 아키텍처는 다음과 같아요.

### 네트워크 계층

- **VPC**: 단일 VPC로 모든 리소스 통합 관리
- **서브넷**: 3개 AZ에 걸친 퍼블릭/프라이빗 서브넷 구성
- **NAT Gateway**: 각 AZ별 NAT Gateway 배치로 최대 고가용성 확보
- **Route53**: 도메인 관리 및 DNS 라우팅

### 컴퓨팅 계층

- **EKS**: Kubernetes 기반 컨테이너 오케스트레이션
- **EC2**: t3.medium 인스턴스 기반 노드 그룹 (3~4개 노드)
- **ALB**: Application Load Balancer로 트래픽 분산

### 데이터베이스 계층

- **RDS (Primary)**: 모든 서비스가 공유하는 메인 데이터베이스
- **RDS (Batch)**: 배치 작업 전용 데이터베이스

### 스토리지 계층

- **S3**: 이미지 및 정적 파일 저장소

### 보안 계층

- **IAM**: 최소 권한 원칙 기반 권한 관리
- **cert-manager**: Let's Encrypt 기반 TLS 인증서 자동 관리
- **Linkerd**: 서비스 메시 기반 mTLS 자동 암호화 및 트래픽 관제

### 모니터링 계층

- **Prometheus**: 메트릭 수집 및 저장 (EKS 클러스터 내 배포)
- **Grafana**: 메트릭 시각화 및 대시보드 (EKS 클러스터 내 배포)

## 아키텍처 설계 철학

저희가 인프라를 설계할 때 가장 중요하게 생각한 원칙들이에요:

1. **최대 고가용성**: 금융 서비스의 높은 가용성 요구사항을 충족하기 위해 최대한 모든 계층에서 고가용성 구성
2. **운영 단순성**: 1명이 무리 없이 관리할 수 있는 수준의 복잡도 유지
3. **비용 최적화**: 예산 제약 내에서 최대한의 성능과 안정성 확보

## 상세 구성 요소별 설계 결정

### 1. VPC 및 네트워크 구성

#### 주요 구성

| 구성 요소         | 설정값                                                          |
| ----------------- | --------------------------------------------------------------- |
| VPC CIDR          | 10.0.0.0/16                                                     |
| 퍼블릭 서브넷     | 3개 (각 AZ별): 10.0.1.0/24, 10.0.2.0/24, 10.0.3.0/24            |
| 프라이빗 서브넷   | 3개 (각 AZ별): 10.0.10.0/24, 10.0.11.0/24, 10.0.12.0/24         |
| 인터넷 게이트웨이 | 1개                                                             |
| NAT 게이트웨이    | 3개 (각 AZ별)                                                   |
| 가용 영역         | 3개 AZ 사용 (ap-northeast-2a, ap-northeast-2b, ap-northeast-2c) |
| 배치 전략         | 각 AZ별 독립 배치로 고가용성 확보                               |
| 라우팅            | 크로스 AZ 라우팅으로 장애 시 백업 경로 제공                     |

#### 설계 결정 근거

- **단일 VPC**: [이전 포스팅](../2025-05-23/infrastructure-stack-selection.md)에서 언급했듯이, Kubernetes 네임스페이스를 통한 논리적 격리로 충분하다고 판단했어요.
- **3개 AZ 사용**: [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_fault_isolation_multiaz_region_system.html)에서 권장하는 고가용성 구성이에요. (최소 2개 AZ 이상)
- **CIDR 설계**: 10.0.0.0/16으로 충분한 확장성을 제공하면서, 서브넷별로 명확한 구분이 가능해요.
- **NAT Gateway 고가용성 구성**: 각 AZ별로 NAT Gateway를 배치하여 최대한의 고가용성을 확보했어요. 한 AZ의 NAT Gateway에 장애가 발생해도 다른 AZ의 프라이빗 서브넷들이 정상 동작하는 NAT Gateway를 통해 계속 인터넷에 접근할 수 있도록 크로스 AZ 라우팅을 구성했어요.
- **NAT Gateway 선택 이유**: [AWS 공식 문서](https://docs.aws.amazon.com/ko_kr/vpc/latest/userguide/vpc-nat-comparison.html)에 따르면, NAT Gateway는 최대 100Gbps까지 자동 확장되어 트래픽 급증에도 안정적으로 대응할 수 있고, [Amazon VPC NAT Gateway SLA](https://d1.awsstatic.com/legal/vpc-sla/Amazon%20VPC%20NAT%20Gateway%20SLA_Korean_2022-05-05.pdf)에 따라 99.9% SLA가 보장되어 서비스 중단 위험을 최소화할 수 있어요.

### 2. EKS 클러스터 구성

#### 주요 구성

| 구성 요소           | 설정값                                                   |
| ------------------- | -------------------------------------------------------- |
| Kubernetes 버전     | 1.32 (AWS EKS 기본값)                                    |
| 노드 그룹           | t3.medium 인스턴스 (3~4개 노드)                          |
| 클러스터 로깅       | API, Audit, Authenticator, Controller Manager, Scheduler |
| 엔드포인트          | 퍼블릭/프라이빗 모두 활성화                              |
| Prometheus 스토리지 | EBS gp3 볼륨 (50GB), 7일 보관                            |
| Grafana 스토리지    | EBS gp3 볼륨 (20GB)                                      |

#### 설계 결정 근거

- **Kubernetes v1.32**: AWS EKS 클러스터 생성 시 기본값인 1.32를 선택했어요.
  - **표준 지원**: 1.32는 2026년 3월 23일까지 표준 지원이 보장되어 있어 추가 비용 없이 안정적으로 사용할 수 있어요.
  - **최신 기능**: 최신 Kubernetes 기능과 보안 패치를 활용할 수 있어요.
  - **AWS 권장**: AWS에서 새 클러스터 생성 시 기본적으로 제공하는 버전으로, AWS가 가장 안정적이라고 판단한 버전이에요.
  - **Add-on 호환성**: [cert-manager](https://cert-manager.io/docs/releases/), [Linkerd](https://linkerd.io/2-edge/reference/k8s-versions/), [ArgoCD](https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/) 등 주요 Add-on들이 모두 1.32를 지원해요.
- **t3.medium 인스턴스 3개**: [이전 포스팅](../2025-06-01/aws-eks-instance-spec-decision.md)에서 분석한 GCP 리소스 사용량을 바탕으로 결정했어요.
- **클러스터 로깅 활성화**: API, Audit, Authenticator, Controller Manager, Scheduler 로깅을 활성화 했어요.
  - **API 로깅**: 모든 kubectl 명령어와 API 호출을 추적해서 보안 감사와 디버깅에 필수적이에요.
  - **Audit 로깅**: 금융 서비스 특성상 모든 작업에 대한 감사 추적이 필요해요. 누가, 언제, 무엇을 했는지 기록해요.
  - **Authenticator 로깅**: 인증 실패나 권한 관련 문제를 추적하기 위해 필요해요.
  - **Controller Manager & Scheduler 로깅**: Pod 스케줄링 문제나 컨트롤러 이슈를 디버깅할 때 필수적이에요.
- **퍼블릭/프라이빗 엔드포인트 모두 활성화**:
  - **퍼블릭 엔드포인트**: 개발자들이 로컬에서 kubectl로 클러스터에 접근할 수 있어요. 보안 그룹으로 특정 IP만 허용해요.
  - **프라이빗 엔드포인트**: 클러스터 내 노드들이 API 서버와 통신할 때 프라이빗 네트워크를 사용해서 보안과 성능을 향상시켜요.
- **모니터링 스택 볼륨 구성**:
  - **Prometheus EBS 볼륨 (50GB)**: Kubernetes PersistentVolume으로 메트릭 데이터 영속성을 확보해요. 7일간 메트릭 저장을 위해 충분한 용량을 확보했어요.
  - **Grafana EBS 볼륨 (20GB)**: 대시보드 설정과 사용자 데이터를 저장하기 위한 영구 스토리지에요. Pod 재시작 시에도 설정이 유지되어요.
  - **gp3 선택 이유**: gp2 대비 20% 더 나은 성능과 비용 효율성을 제공해요. IOPS와 처리량을 독립적으로 조정할 수 있어 모니터링 워크로드에 최적화되어 있어요.

### 3. RDS 구성

#### 주요 구성

| 구성 요소   | 메인 데이터베이스              | 배치 데이터베이스                    |
| ----------- | ------------------------------ | ------------------------------------ |
| 엔진        | MySQL 8.0                      | MySQL 8.0                            |
| 인스턴스    | db.t3.small (2 vCPU, 2GB RAM)  | db.t3.small (2 vCPU, 2GB RAM)        |
| 스토리지    | 30GB gp2, 최대 100GB 자동 확장 | 30GB gp2                             |
| 백업        | 1일 보관, 새벽 3-4시 자동 백업 | 1일 보관, 새벽 2-3시 자동 백업       |
| 암호화      | 저장 시 암호화 활성화          | 저장 시 암호화 활성화                |
| 배치        | 단일 AZ 인스턴스 배포          | 단일 AZ 인스턴스 배포                |
| 연결 서비스 | 메인 서비스 (총 15개 Pod)      | 배치 서비스 (총 9개 Pod)             |
| 용도        | 실시간 서비스 처리             | 배치 작업 전용으로 메인 DB 부하 분산 |

#### 설계 결정 근거

- **DB 분리**: 메인 DB와 배치 DB를 분리해서 배치 작업이 실시간 서비스에 영향을 주지 않도록 했어요.
- **실제 DB 연결 분석**: [이전 분석](../2025-06-01/aws-eks-instance-spec-decision.md)에서 76개 Pod 중 실제 DB에 연결하는 서비스는 다음과 같아요:
  - **메인 DB 연결**: 총 15개 Pod
  - **배치 DB 연결**: 총 9개 Pod
  - 나머지 52개 Pod는 프론트엔드, 모니터링, 시스템 컴포넌트로 DB 연결이 불필요해요.
- **메인 DB 스펙 결정**: 15개 Pod가 연결하고 각 Pod당 [HikariCP 기본 풀 크기](../2025-05-26/mysql-max-connection-troubleshooting.md) 10개를 고려하면 최대 150개 연결이 필요하므로, db.t3.small(2 vCPU, 2GB RAM)로 충분한 연결 처리 능력을 확보했어요.
- **배치 DB 스펙 결정**: 9개 Pod가 연결하고 각 Pod당 HikariCP 기본 풀 크기 10개를 고려하면 최대 90개 연결이 필요하므로, db.t3.small(2 vCPU, 2GB RAM)로 안정적인 연결 처리 능력을 확보했어요.
- **단일 AZ 선택 이유**: AWS RDS 고가용성 템플릿의 현실적 제약사항을 고려한 결정이에요:

**AWS RDS 고가용성 템플릿의 비용 제약사항:**

AWS는 RDS 고가용성을 위해 다음과 같은 템플릿을 제공하지만, 모두 예산 제약을 초과해요:

1. **Multi-AZ DB 인스턴스 배포 (인스턴스 2개)**:

   - 최소 스펙: `db.m5.large` (2 vCPU, 8GB RAM)
   - 최소 스토리지: 20GiB gp2
   - 월별 비용: **최소 $349.80**
   - 특징: 동기식 복제, 자동 페일오버 (60초 이내)

2. **Multi-AZ DB 클러스터 배포 (인스턴스 3개)**:
   - 최소 스펙: `db.m5.large` × 3개 (6 vCPU, 24GB RAM 총합)
   - 최소 스토리지: 20GiB gp2
   - 월별 비용: **최소 $594.78**
   - 특징: Aurora 기반, 읽기 복제본 포함, 더 빠른 복구 시간

**비용 분석 (10일 운영 기준):**

- Multi-AZ 인스턴스: $349.80 ÷ 3 = **$116.6** (10일 기준)
- Multi-AZ 클러스터: $594.78 ÷ 3 = **$198.3** (10일 기준)
- 현재 단일 AZ: $16.4 (10일 기준)

**예산 제약 분석:**

- 총 예산: 25만원 (약 $181, 환율 1,380.44원 기준)
- Multi-AZ 최소 비용: $116.6 (DB만으로 예산의 64% 소모)
- Multi-AZ 클러스터 비용: $198.3 (DB만으로 예산의 110% 초과)
- 나머지 인프라 비용: $65+ (EKS, EC2, NAT Gateway 등)
- **결과**: Multi-AZ 사용 시 총 예산 $181 대폭 초과 불가피

**가용성 대안 전략:**

- **애플리케이션 레벨 장애 복구**: HikariCP 연결 풀의 자동 재연결 기능 활용
- **데이터 백업**: 일일 자동 백업으로 데이터 손실 방지 (RPO 24시간)
- **모니터링 강화**: Prometheus + Grafana로 DB 상태 실시간 모니터링
- **빠른 복구**: RDS 스냅샷 기반 신속한 인스턴스 복구 (RTO 30분 이내)

- **1일 백업**: 10일간 운영이지만 금융 서비스 특성상 최소한의 백업 정책 유지
- **MySQL 8.0 선택 이유**: MySQL 5.7과 8.4 대신 8.0을 선택한 구체적인 이유는 다음과 같아요:
  - **장기 안정성**: MySQL 8.0은 **2026년 7월 31일까지 AWS RDS 표준 지원**이 보장되어 있어요. 5.7은 이미 2024년 2월 29일에 표준 지원이 종료되어 추가 지원 요금이 발생하고, 8.4는 2024년 4월 출시로 아직 검증 기간이 짧아요.
  - **최신 마이너 버전**: **MySQL 8.0.41** (2025년 2월 19일 AWS RDS 릴리즈)을 사용할 수 있어 최신 보안 패치와 안정성을 확보해요. tzdata2025a 기반 시간대 정보와 RDS 저장 프로시저 관련 버그 수정이 포함되어 있어요.
  - **성능 향상**: 8.0은 5.7 대비 읽기/쓰기 성능이 약 2배 향상되었고, JSON 데이터 타입 처리 성능도 크게 개선되었어요.
  - **금융 서비스 특화 보안**: SHA-256 기반 caching_sha2_password 인증, 역할 기반 권한 관리(CREATE ROLE, GRANT ROLE), 데이터 마스킹 등이 강화되어 금융 데이터 보호에 적합해요.
  - **개발 효율성**:
    - **JSON 타입**: 주문 상세 정보나 결제 메타데이터를 JSON으로 저장할 때 네이티브 지원으로 쿼리 성능과 개발 생산성이 향상되어요.
    - **CTE (Common Table Expression)**: 복잡한 재귀 쿼리나 계층적 데이터 처리에 유용해요.
    - **Window Function**: ROW_NUMBER(), RANK(), SUM() OVER() 등으로 매출 분석이나 순위 쿼리를 효율적으로 작성할 수 있어요.
  - **글로벌 대응**: UTF8MB4가 기본 문자셋으로 설정되어 이모지나 다국어 지원이 원활해요.
  - **비용 효율성**: 5.7은 추가 지원 요금($0.025/시간)이 발생하지만, 8.0은 표준 지원으로 추가 비용이 없어요.
- **암호화**: 금융 데이터 보호를 위해 저장 시 암호화를 활성화했어요.

### 4. S3 구성

#### 주요 구성

| 구성 요소          | 설정값                       |
| ------------------ | ---------------------------- |
| 버저닝             | 활성화                       |
| 서버 사이드 암호화 | 활성화                       |
| 퍼블릭 액세스      | 허용 (이미지 파일 접근 목적) |

#### 설계 결정 근거

- **버저닝 활성화**: 실수로 파일을 덮어쓰거나 삭제했을 때 복구할 수 있어요.
- **암호화**: 모든 데이터를 암호화해서 저장해요.
- **퍼블릭 액세스 허용**: 원큐 오더 서비스에서 매장 이미지 사진이나, 메뉴 이미지 접근을 위해 퍼블릭 엑세스를 허용했어요.

### 5. IAM 구성 (최소 권한 원칙)

**주요 정책 구성**

**개발자 그룹 (Developer Group)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster",
        "eks:ListClusters",
        "eks:DescribeNodegroup",
        "eks:ListNodegroups"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": ["arn:aws:s3:::wonq-images/*", "arn:aws:s3:::wonq-images"]
    }
  ]
}
```

**EKS 서비스 계정 (Service Account)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["rds:DescribeDBInstances", "rds:DescribeDBClusters"],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::wonq-images/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets",
        "route53:GetChange",
        "route53:ListResourceRecordSets"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "route53:ChangeAction": ["UPSERT", "DELETE"]
        }
      }
    }
  ]
}
```

**모니터링 서비스 계정**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudwatch:GetMetricStatistics",
        "cloudwatch:ListMetrics",
        "ec2:DescribeInstances",
        "eks:DescribeCluster",
        "rds:DescribeDBInstances"
      ],
      "Resource": "*"
    }
  ]
}
```

**설계 결정 근거:**

- **최소 권한 원칙**: 각 역할별로 꼭 필요한 권한만 부여해요.
- **서비스 계정 분리**: EKS 내부 서비스들은 Kubernetes Service Account와 IAM Role을 연결해서 세밀한 권한 제어를 해요.
- **리소스 제한**: S3는 특정 버킷만, Route53은 특정 액션만 허용해요.
- **모니터링 권한**: Prometheus와 Grafana가 AWS 리소스 메트릭을 수집할 수 있도록 읽기 전용 권한을 부여해요.

### 6. ALB (Application Load Balancer) 구성

**주요 구성:**

| 구성 요소 | 설정값                          |
| --------- | ------------------------------- |
| ALB 타입  | Application Load Balancer       |
| 스키마    | 인터넷 연결형 (Internet-facing) |
| 타겟 그룹 | EKS 워커 노드들                 |
| 헬스 체크 | HTTP/HTTPS 헬스 체크 활성화     |
| 보안 그룹 | HTTP(80), HTTPS(443) 포트 허용  |

**설계 결정 근거:**

- **ALB 선택**: Layer 7 로드 밸런싱으로 HTTP/HTTPS 트래픽에 최적화되어 있어요.
- **인터넷 연결형**: 외부 사용자가 접근할 수 있도록 퍼블릭 서브넷에 배치해요.
- **타겟 그룹**: Kubernetes Service와 연동하여 자동으로 타겟 관리해요.

### 7. Route53 구성

**주요 구성:**

| 구성 요소       | 설정값                                                                |
| --------------- | --------------------------------------------------------------------- |
| 도메인          | wonq.store (기본 도메인)                                              |
| 서브도메인 설정 | api.wonq.store 외 11개 서브도메인 (사용자 앱, 가맹점 앱, 모니터링 등) |
| A 레코드        | ALB와 연결                                                            |
| 헬스 체크       | ALB 상태 모니터링                                                     |

### 8. 비용 분석

**10일간 운영 비용**을 분석해보면 다음과 같아요 (ap-northeast-2 서울 리전 기준)

| 리소스                | 인스턴스/설정      | 시간당 비용 | 10일 비용 (USD) | 10일 비용 (KRW)  |
| --------------------- | ------------------ | ----------- | --------------- | ---------------- |
| EKS 클러스터          | 클러스터 제어 평면 | $0.10       | $24             | ₩33,130.56       |
| EC2 (노드)            | t3.medium × 3      | $0.1384     | $33.22          | ₩45,857.368      |
| RDS 메인              | db.t3.small        | $0.0416     | $9.98           | ₩13,774.912      |
| RDS 배치              | db.t3.small        | $0.0416     | $9.98           | ₩13,774.912      |
| NAT Gateway           | 3개 (각 AZ)        | $0.059      | $42.48          | ₩58,634.112      |
| Elastic IP            | 3개 (NAT용)        | $0.005      | $3.6            | ₩4,969.584       |
| ALB                   | 1개 (3 AZ 배치)    | $0.0243     | $5.83           | ₩8,047.951       |
| Route53 헬스 체크     | 12개 서브도메인    | $0.75/월    | $30.0           | ₩41,413.2        |
| EBS (Prometheus)      | gp3 50GB           | $0.096/월   | $3.2            | ₩4,417.408       |
| EBS (Grafana)         | gp3 20GB           | $0.096/월   | $1.28           | ₩1,766.963       |
| Route53               | 호스팅 존          | $0.50/월    | $1.67           | ₩2,305.335       |
| S3                    | 기본 사용량        | -           | $2.5            | ₩3,451.1         |
| 기타 (데이터 전송 등) | -                  | -           | $4              | ₩5,521.76        |
| **총합**              |                    |             | **$171.74**     | **₩236,862.462** |

_6월 2일 오전 12:09분 기준, 환율: 1달러 = 1,380.44원_

**주요 가격 차이점 (서울 리전 vs 버지니아 리전):**

- **EC2**: t3.medium이 서울에서 약 11% 더 비싸요 ($0.1248 → $0.1384)
- **RDS**: db.t3.small이 서울에서 약 22% 더 비싸요 ($0.034 → $0.0416)
- **NAT Gateway**: 데이터 처리 비용이 서울에서 약 67% 더 저렴해요 ($0.059 vs $0.177, 1GB당)
- **Route53 헬스 체크**: 서울에서 50% 더 비싸요 ($0.50 → $0.75/월)
- **EBS**: gp3 스토리지가 서울에서 20% 더 비싸요 ($0.08 → $0.096/GB/월)

## 참고 자료

- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [Amazon EKS User Guide](https://docs.aws.amazon.com/eks/latest/userguide/)
- [Amazon RDS User Guide](https://docs.aws.amazon.com/rds/latest/userguide/)
- [NAT 게이트웨이 및 NAT 인스턴스 비교](https://docs.aws.amazon.com/ko_kr/vpc/latest/userguide/vpc-nat-comparison.html)
- [AWS Pricing Calculator](https://calculator.aws/)
- [Kubernetes 공식 문서](https://kubernetes.io/docs/home/)
- [Send control plane logs to CloudWatch Logs](https://docs.aws.amazon.com/eks/latest/userguide/control-plane-logs.html#control-plane-logs-types)
- [Control network access to cluster API server endpoint](https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html)
