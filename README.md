# 💳 Payment Reconciliation & Monitoring Pipeline
> **n8n과 ELK Stack을 활용한 결제 승인 알림 및 자동 대사 관제 시스템**

## 📖 Project Overview
본 프로젝트는 가맹점 결제 시스템의 안정성과 데이터 정합성을 보장하기 위해 구축된 **End-to-End 결제 데이터 파이프라인**입니다.

실시간으로 발생하는 결제 건에 대해 **Discord 알림**을 발송하여 즉각적인 모니터링 체계를 구축하고, 매일 발생하는 VAN사 배치 파일(CSV)과 내부 원장(DB)을 자동으로 대조(Reconciliation)하여 자금 흐름의 누락이나 불일치를 시각화합니다.

## 🚀 Key Features

### 1. High-Value Transaction Monitoring (Front-end)
* 실시간으로 발생하는 결제 트랜잭션을 모니터링합니다.
* 설정된 임계값(Threshold)을 초과하는 고액 결제나 이상 패턴 발생 시, **Discord Webhook**을 통해 관리자에게 즉시 경고 알림을 발송합니다.

### 2. Automated Verification & ETL Pipeline (Middle-ware)
* **가맹점 결제 기록**과 **카드사 결제 기록**을 수집하여 자동으로 비교 검증합니다.
* 양쪽 데이터가 일치하는 **검증된 거래**만을 선별하여 분석용 DB 및 Elasticsearch에 적재합니다.
* 데이터 불일치 발생 시 별도 로그로 분류하여 정합성을 보장합니다. (이 부분 수정해주세요)

### 3. Customer Spending Analytics (Back-end)
* 검증이 완료된 신뢰도 100%의 데이터를 기반으로 고객 분석을 수행합니다.
* **Kibana**를 통해 고객별 소비 비율, 시간대별 지출 패턴, 카테고리별 선호도 등을 시각화합니다.

## 🛠 System Architecture (workflow 사진 첨부)

### 1. 실시간 모니터링 및 알림 (Real-time Monitoring)
<img width="1730" height="847" alt="image" src="https://github.com/user-attachments/assets/33b145a7-2f8b-4053-8524-6235fcbafe4e" />

## 🧰 Tech Stack

| Category | Technology | Usage |
| :--- | :--- | :--- |
| **Workflow** | **n8n** | 고액 결제 필터링, 데이터 비교/검증 로직 자동화 |
| **Database** | **MySQL** | 가맹점 및 카드사 원장 데이터 관리 |
| **Log/Search** | **Elasticsearch (7.17)** | 검증된 분석용 데이터셋 적재 (Security Enabled) |
| **Visualization** | **Kibana** | 고객 소비 패턴 시각화 대시보드 |
| **Alerting** | **Discord** | 고액/이상 결제 실시간 관리자 알림 |
| **Infra** | **Docker** | 컨테이너 기반 환경 구성 |

## 📂 Workflow Logic Details

### Part 1. High-Value Alerting (Front-end Logic)
* **Input**: 실시간 결제 트랜잭션 (DB Polling/Webhook)
* **Logic**: `If Node`를 사용하여 결제 금액이 설정된 기준(예: 1,000,000원) 이상인지 판단
* **Action**: 조건 충족 시 **Discord 관리자 채널**로 상세 결제 정보(금액, 가맹점, 시간) 즉시 전송

### Part 2. Data Verification (Cross-Validation Pipeline)
* **Process**:
    1.  **Merge**: 가맹점 원장(DB)과 카드사 내역(CSV)을 `Transaction ID` 기준으로 병합
    2.  **Validation**: 결제 금액 및 승인 번호의 양측 일치 여부 확인
    3.  **Loading**: 검증에 성공한 데이터만 최종 분석용 인덱스(`verified-payment-data`)에 적재

## 🔧 Trouble Shooting

### 1. Elasticsearch 7.x 보안 모듈 수동 활성화
* **Issue**: ES 7.17 버전은 8.x와 달리 보안(X-Pack Security)이 기본 비활성화되어 있어 외부 접속 제어가 불가능함.
* **Resolution**:
    1.  컨테이너 내부 `elasticsearch.yml` 파일을 호스트로 복사 (`docker cp`).
    2.  `xpack.security.enabled: true` 옵션 수동 추가.
    3.  `elasticsearch-setup-passwords interactive` 툴을 통해 `elastic`, `kibana_system` 등 계정별 비밀번호 설정 완료.

### 2. 저사양 환경(4GB RAM)에서의 메모리 최적화
* **Issue**: 4GB 메모리 환경에서 ELK Stack과 n8n 동시 구동 시 OOM(Out Of Memory)으로 컨테이너 강제 종료 발생.
* **Resolution**:
    * **Swap Memory**: `fallocate`를 사용하여 4GB Swap 파일 생성 및 마운트하여 부족한 메모리 확보.
    * **Heap Size**: Elasticsearch JVM Heap 사이즈를 환경에 맞게 제한 설정.
      
## 👥 Contributors

| Name | Role | GitHub |
| :--- | :--- | :--- |
| **Name 1 (Leader)** | PM & Workflow Design | [GitHub Profile](https://github.com/janie71) |
| **Name 2** | ELK Stack Infrastructure | [GitHub Profile](https://github.com/...) |
| **Name 3** | Database & Query Optimization | [GitHub Profile](https://github.com/janie71) |
