# 💳 n8n을 이용한 고객 카드 결제 내역 DB 적재 파이프라인 자동화
> CSV 거래내역을 적재하고, ES/Kibana로 사용내역을 시각화하며, 적재 결과를 푸시로 알리는 자동화 파이프라인

## 👥 Contributors
|       배기영       |       류승환        |      이유진        |
| :-----------------: | :-----------------: | :----------------: |
| [<img width="160px" src="https://github.com/bbky323.png">](https://github.com/bbky323) | [<img width="160px" src="https://github.com/Federico-15.png">](https://github.com/Federico-15) | [<img width="160px" src="https://github.com/janie71.png">](https://github.com/janie71) |
| [@bbky323](https://github.com/bbky323) | [@Federico-15](https://github.com/Federico-15) | [@janie71](https://github.com/janie71) |

## 🧰 Tech Stack

| Category | Technology | Usage |
| :--- | :--- | :--- |
| **Workflow** | **n8n** | 고액 결제 필터링, 데이터 비교/검증 로직 자동화 |
| **Database** | **MySQL** | 가맹점 및 카드사 원장 데이터 관리 |
| **Log/Search** | **Elasticsearch (7.17)** | 검증된 분석용 데이터셋 적재 (Security Enabled) |
| **Visualization** | **Kibana** | 고객 소비 패턴 시각화 대시보드 |
| **Alerting** | **Discord** | 고액/이상 결제 실시간 관리자 알림 |
| **Infra** | **Docker** | 컨테이너 기반 환경 구성 |

## 📖 Project Overview
 **n8n**으로 카드 승인 거래 CSV **배치 파일을 자동으로 읽어 DB에 적재**하는 ETL 파이프라인을 구현했다. 데이터는 정제/검증 후 SHA256 기반 txn_key를 생성해 중복을 방지하며, **스테이징(transactions_stage)과 정본(transactions)을 분리**해 운영 구조를 모사했다. 적재 단위별 run_id와 ingest_runs 로그로 **시작/종료 상태 및 처리 건수를 기록**하고, 적재 결과를 Discord로 푸시 알림하여 **모니터링을 강화**했다. 또한 **Elasticsearch와 Kibana**를 연동해 카드 사용 내역을 기간/카테고리/가맹점 관점에서 **통계적으로 시각화**했다.

## 🚀 Key Features
* 배치 파일 적재 자동화: 지정 폴더(/opt/batch/incoming)의 CSV를 읽어 거래 row 파싱
* 정제/검증: 타입 변환(금액 int, boolean 등), 공백/NULL 처리, 기본값 보정
* 중복 방지 키 생성: 거래 식별용 txn_key를 SHA256 해시로 생성
* 스테이징 적재: transactions_stage에 저장 + run_id로 적재 단위 추적
* 정본 UPSERT: transactions에 txn_key 기준 Insert or Update로 중복 방지
* 적재 로그 관리: ingest_runs에 적재 시작/종료 상태 및 처리 건수 기록(STARTED → DONE)
* Discord 푸시 알림: 적재 성공/실패 및 처리 건수(총 건수, 성공/실패)를 Discord 채널로 자동 전송
* ES/Kibana 시각화: 적재된 카드 사용 내역을 Elasticsearch에 적재/동기화하고, Kibana 대시보드로 기간별 소비, 카테고리별 지출, 가맹점 TOP N 등 통계 시각화

## 📂 Project Background
### 발단
* 수업에서 제공된 데이터셋을 **DB에 반복적으로 적재**하는 과정이 생각보다 번거롭고(전처리·타입 변환·중복 처리·에러 확인 등) 시간이 많이 든다는 걸 체감했다.
* “**적재 과정을 자동화**하면 생산성이 확 올라가겠다”는 문제의식이 생겼고, 단순 자동화가 아니라 **실무에서 실제로 쓰는 형태**와 비슷하게 만들어 보고 싶었다.
* 그래서 **ETL(Extract → Transform → Load)** 흐름을 갖춘 파이프라인을 직접 설계·구현하는 것을 목표로 잡았다.
### 확장
* 실무 연관성을 찾다 보니, 금융/카드 도메인에서는 카드 결제/정산 데이터가 실시간 API만으로 끝나지 않고, **배치형(CSV/파일)**으로 “모아서** 처리**”되는 경우가 많다는 점이 눈에 들어왔다.
* 실제 결제 흐름을 배치 관점으로 단순화한 예시
 <img src="images\visualselection.png">
### 결론
* 단순히 “CSV를 DB에 넣기”가 아니라, 실무에서 흔한 배치 데이터 흐름을 모델로 삼아 적재(run) 단위 추적 → 원본 스테이징 적재 → 1·2차 정제/대사 → 정본 적재 → 시각화/알림까지 이어지는 파이프라인을 자동화해보기로 했다.

## 📂 Workflow Logic Details
<img src="images\n8nflow1.png">
* 배치 파일(csv 데이터셋)을 읽어온 다음, 각 거래 건마다 고유키 생성 및 개인정보 해시
* 배치 파일 원본 적재
* 배치 파일 원본 적재 후 중복값 등 데이터 1차 정제
<img src="images\n8nflow2.png">
* 1차 정제한 배치파일 원본과 현장 결제 승인/거절 데이터와 비교하여 2차 검증
* 2차 검증 후 데이터 정본을 DB에 적재
<img src="images\n8nflow3.png">
* 정본 적재와 동시에 성공/실패 여부를 푸시
* 정본 적재 후 데이터 및 ES와 kibana를 이용하여 사용자 통계 분석

## 🎁 Result


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
      
