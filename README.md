# ETL_piepline

# 🚀 데이터 파이프라인 프로젝트

> Object Storage에 적재되어 있는 Trail Event 데이터를 분석/집계하여 MySQL에 저장하는 Java 기반 ETL 파이프라인 구현

<br>

## 🗓️ 주차별 진행 상황

### ✅ 1주차

- 현재 시간을 출력하는 애플리케이션 작성 및 Gradle을 통한 JAR 파일 빌드
- Docker 이미지 생성 후 카카오클라우드 컨테이너 레지스트리(KC CR)에 이미지 PUSH 완료

### ✅ 2주차

- 카카오클라우드 Object Storage에 저장된 로그 압축 파일(`.gz`) 다운로드 기능 구현
- 메모리 상에서 스트림을 통해 압축 파일 해제 및 텍스트 읽기 로직 구현

### ✅ 3주차

- **🎯 프로젝트 목표:** 추출된 데이터를 적재할 데이터 마트(MySQL) 스키마를 구축하고, 다운로드한 원본 데이터와 집계 지표를 저장한다.
- **주요 작업:** Java Application 내에서 데이터베이스 연결 및 테이블 스키마 초기화 로직(`DatabaseInitializer`) 구현

> 🚨 **[중요: 빌드 시 주의사항]** 🚨
>
> 본 프로젝트의 테스트 코드(`DatabaseInitializerTest`)는 **실제 로컬 DB(Docker MySQL) 환경**과의 통신을 요구합니다.
> 도커가 실행 중이지 않은 상태에서 일반 빌드를 시도할 경우, 테스트 실패로 인해 **빌드 에러가 발생**합니다.
>
> **도커를 실행하지 않고 코드를 빌드하려면, 아래 명령어를 사용하여 테스트 검사 단계를 제외해 주세요.**
> ```bash
> $ ./gradlew build -x test
> ```

### ✅ 4주차

#### 📊 도출 지표 설명

1. **시간대별 서비스별 호출 빈도**

- **설명:** 지정된 날짜(date)의 24시간 동안 발생한 로그를 기반으로,각 서비스(event_source)별 시간당 이벤트 호출 수 집계
- **도출 방법:**
   * WHERE DATE(event_time) = ? 조건으로 해당 날짜의 데이터 필터링
   * EXTRACT(HOUR FROM event_time)로 시간 정보 추출한 뒤, event_source 컬럼과 함께 GROUP BY 처리
   * COUNT(*) 집계 함수를 사용하여 시간대별, 서비스별 이벤트 호출 건수(service_call_count) 도출

- **적재 위치:** : MySQL 데이터 마트 테이블(service_call)
- **테이블 PK** : real_date, real_hour, event_source 3개 컬럼을 복합 기본키로 구성


2. **시간대별 트래픽**
   - **설명:** 지정된 날짜(date)의 24시간동안 발생한 로그를 기반으로, 서비스 구분 없이 전체에서 발생한, 시간당 총 이벤트(API 호출 등) 건수를 집계
   - **도출 방법:**
      * WHERE DATE(event_time) = ? 조건으로 해당 날짜의 데이터 필터링
      * EXTRACT(HOUR FROM event_time)로 시간 정보 추출한 뒤, 해당 시간을 기준으로 GROUP BY 처리
      * COUNT(*) 집계함수를 사용하여 시간당 총 트래픽 건수(traffic_count)를 도출하고, 시간순(ORDER BY HOUR)으로 정렬

   - **적재 위치** : MySQL 데이터 마트 테이블(traffic)
   - **테이블 PK** : real_date, real_hour


3. **DAU**
   - **설명:** 지정된 날짜(date)의 24시간동안, 각 서비스(event_source)를 이용한 순수 방문자 수(중복 제거) 집계
   - **도출 방법**
      * WHERE DATE(event_time) = ? 조건으로 해당 날짜의 로그 데이터 필터링
      * event_source 컬럼으로 GROUP BY 처리
      * COUNT(DISTINCT user_id) 집계 함수를 사용하여 동일 사용자의 중복 방문을 제거하고, 순수 방문자 수(dau_count) 도출
   - **적재 위치** : MySQL 데이터 마트 테이블(dau)
   - **테이블 PK** : real_date, event_source


4. **시간대별 데이터 쿼리서비스 쿼리 실행 성공률** :
   - **설명:** 지정된 날짜(date)의 24시간동안, 'Data Query' 서비스에서 실행된 전체 쿼리(Query) 중 성공적으로 완료된 쿼리의 비율(%)을 시간대별로 집계
   - **도출 방법**
      * WHERE DATE(event_time) = ? AND event_source = 'Data Query' AND resource_type = 'Query' 조건으로 서비스와 날짜 필터링
      * EXTRACT(HOUR FROM event_time) 함수로 시간 추출한 뒤, 해당 시간(HOUR)으로 GROUP BY 처리
      * SUM(CASE WHEN event_name = 'Query Execution Success' THEN 1 ELSE 0 END) 를 통해 성공 건수만 분자로 삼고, 전체 건수(COUNT(*))로
        나누어 백분율(*100.0) 계산
      * ROUND() 함수를 적용해 최종 성공률(query_success_rate)을 소수점 둘째 자리까지 반올림하여 도출
   - **적재 위치** : MySQL 데이터 마트 테이블(query_success_rate)
   - **테이블 PK** : real_date, real_hour


5. **스스로 생각한 지표** : 일일 사용자 활동량 랭킹 TOP 10
   - **설명:** 지정된 날짜(date)의 24시간 동안 가장 많은 이벤트를 발생시킨 상위 10명의 활동량을 집계(단, 유저가 불분명한 케이스는 제외)
   - **도출 방법**
      * WHERE DATE(event_time) = ? 조건으로 해당 날짜의 로그 데이터 필터링
      * 유저가 불분명한 케이스를 순위에서 제외하기 위해 AND user_name != '-' 조건 추가
      * user_name 컬럼으로 GROUP BY 처리
      * COUNT(*) 집계 함수를 사용하여 각 사용자별 총 이벤트 발생 건수(user_activity_count) 도출
      * ORDER BY user_activity_count DESC LIMIT 10 구문을 적용하여 활동량이 많은 순서대로(내림차순) 정렬한 뒤, 최상위 10명의 데이터만 추출
   - **적재 위치** : MySQL 데이터 마트 테이블(topuser)
   - **테이블 PK** : real_date, user_name

### ✅ 5주차
- **🎯 프로젝트 목표:** k8s Job을 활용하여 대량의 데이터를 배치 처리(ETL)하고, Superset을 연동하여 시각화 대시보드를 구축한다.

- **주요 작업:** 한 달 치 데이터 적재를 위한 etl-job.yaml 작성 및 실행 , Superset을 활용한 3종(Overview, Monthly, Daily) 대시보드 생성
  <br><br>
- **Overview Dashboard:** 핵심 성과 지표(KPI)와 전체 데이터를 한눈에 파악할 수 있는 통합 계기판
   * 필터 : 서비스 선택, Time range, 사용자 유형 선택(실사용자, 시스템 계정)
![katie-overview-dashboard-2026-06-22T08-38-34 518Z](https://github.kakaoenterprise.in/gc26/katie-project/assets/2944/b43e6b6b-b4d0-435d-bda4-7d2cc9c40984)



  <br><br>
- **Monthly Aggregation Dashboard:** 월별 집계 데이터를 확인하는 대시보드
   * 필터 : 서비스 선택, 연/월 선택, 사용자 유형 선택(실사용자, 시스템 계정)
  ![katie-monthly-aggregation-dashboard-2026-06-22T08-58-10 054Z](https://github.kakaoenterprise.in/gc26/katie-project/assets/2944/a9722886-771a-4eff-989b-2cd37ca05aa6)

  <br><br>
- **Daily Aggregation Dashboard:**
   * 필터 : 서비스 선택, 연/월/일 선택, 사용자 유형 선택(실사용자, 시스템 계정)
  ![katie-daily-aggregation-dashboard-2026-06-22T09-18-08 652Z](https://github.kakaoenterprise.in/gc26/katie-project/assets/2944/24454aea-272a-408b-a4a8-7ac75fb2cc80)


### ✅ 6주차
- **🎯 프로젝트 목표:** 데이터베이스 인프라를 카카오 클라우드 MySQL로 이관하고, K8s CronJob을 도입하여 일일 ETL 파이프라인을 자동화 및 과거 데이터 재수행(Backfill) 전략을 확보

- **주요 작업:** 
   * 개발용 로컬 MySQL 환경을 고가용성이 보장되는 카카오 클라우드 매니지드 MySQL 서비스로 전격 이관 및 연동. DB 접속 정보 및 S3 키 등 민감 데이터는 K8s Secret을 활용해 안전하게 관리하도록 구조화
   * 매일 지정된 시간(KST 오전 11시)에 자동으로 전날 기준의 트레일 이벤트 데이터를 자동으로 집계하도록 etl-job.yaml 작성 및 운영 파이프라인 구축
     * 크론잡 실패 또는 특정 일자 데이터 누락 시, 기존 뼈대를 일회성 job으로 추출한 뒤, 스크립트 환경 변수에 원하는 과거 날짜(DATE)를 직접 주입하여 타겟 일자만 유연하게 복구
<br><br>
   ---
   ## 🛠️ 장애 복구 및 수동 재수행 가이드 (Backfill)
   <br><br>
   K8s CronJob의 특성상 런타임 서버 시간을 기준으로 작업 대상 날짜를 설정합니다. 특정 일자의 크론잡이 실패하였을 때, 수동 재수행 프로세스는 다음과 같습니다.
<br><br>
   ### 수동 재수행 프로세스🔨
  
   **1. 기존 CronJob 기반으로 1회성 Job 뼈대 추출**
   터미널에서 아래 명령어를 실행하여 현재 운영 중인 크론잡 설정을 'manual-job.yaml' 파일로 추출합니다.
  ```bash
  kubectl create job katie-etl-manual-retry --from=cronjob/katie-etl-job -n pipeline --dry-run=client -o yaml > manual-job.yaml
  ```
  <br><br>
   **2. 타겟 날짜(DATE) 환경변수 주입**
   생성된 manul-job.yaml 파일을 열고, env 영역에 재수행할 타겟 날짜를 직접 추가합니다.
    ```yaml
    env:
    - name: DATE
      value: "2026-06-15"
    ```
   <br><br>
    **3. 수동 Job 배포**
    
    ```bash
    kubectl apply -f manual-job.yaml
    ```
     <br>

