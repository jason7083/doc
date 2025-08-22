# RA 시스템 상세 설계안 및 DR 시나리오 샘플

---

## 1. RA 시스템 상세 설계안

### 1.1. 시스템 구성도

```
[뱅킹 채널/외부 시스템]
        │
  (REST API/Socket)
        │
[RA 시스템]
 ├─ API/전문 수신 모듈 (REST/Socket)
 ├─ 전문 파싱/생성 모듈 (설정파일 기반)
 ├─ 암호화/복호화 모듈 (파일기반 키/인증서)
 ├─ 트랜잭션 처리/이력 적재 모듈
 ├─ 장애 감지 및 자동복구 모듈
 ├─ 환경설정/운영 정책 관리 모듈
 └─ DB 연동 모듈 (Oracle)
        │
   [OTP/금융결제원 등 외부 연계]
```

### 1.2. 주요 모듈별 상세

#### 1) API/전문 수신 모듈
- REST API (채널→RA): POST/PUT 방식, GET 미지원
- 전문별 URI/메서드 분리, 인증·암호화 정책은 설정파일로 제어
- 전문 수신 후 파싱 모듈로 전달

#### 2) 전문 파싱/생성 모듈
- 전문 레이아웃은 XML/YML 등 설정파일에서 동적 로드
- 신규 전문 추가/변경 시 설정파일만 수정 (코드 수정 無)
- 전문 항목별 마스킹도 설정 가능

#### 3) 암호화/복호화 모듈
- 파일 기반 암호키/인증서 관리 (AIX 파일시스템 내 관리)
- 채널/구간/전문별 암호화 정책은 config 파일에서 로드
- 암호화/복호화 실패 시 에러로그 및 관리자 알림

#### 4) 트랜잭션 처리/이력 적재 모듈
- 거래별 상세 이력, 전문원문, 응답코드, 사용자정보, 마스킹 필드 등을 DB(Oracle) 적재
- 이력 테이블은 일자/채널/거래유형 등으로 파티셔닝 가능

#### 5) 장애 감지 및 자동복구 모듈
- 프로세스 헬스체크 및 주요 자원 모니터링
- 서비스 비정상 종료 감지 시 자동 재기동 스크립트 연동
- 장애 발생 시 UMS 통한 SMS, SMTP 통한 Email 관리자 통보 (내용·대상은 설정파일 제어)
- 장애·복구 이벤트 DB 적재

#### 6) 환경설정/운영 정책 관리 모듈
- 모든 운영 파라미터와 정책(암호화, 응답코드, 장애대응 등)은 설정파일로 외부화
- 설정파일 변경시 hot-reload 또는 최소 downtime 적용

#### 7) DR/BCP 지원 구조
- Active/Active 이중화(설정파일로 센터별 분기)
- DR 모의훈련 시 별도 환경설정 파일로 신속 전환
- 복구/전환 이력 DB 적재

---

### 1.3. 데이터베이스 테이블 예시

```sql
-- 1. 거래 이력 테이블
CREATE TABLE TRANSACTION_HISTORY (
    TRAN_ID VARCHAR2(32) PRIMARY KEY,
    CHNL_ID VARCHAR2(20),
    MSG_TYPE VARCHAR2(20),
    REQ_MSG CLOB,
    RESP_MSG CLOB,
    RESP_CD VARCHAR2(10),
    USER_ID VARCHAR2(50),
    MASKED_FIELD VARCHAR2(200),
    TX_DT DATE,
    ETC1 VARCHAR2(100),
    ETC2 VARCHAR2(100)
);

-- 2. 장애/복구 이력
CREATE TABLE RECOVERY_EVENT (
    EVENT_ID VARCHAR2(32) PRIMARY KEY,
    EVENT_TYPE VARCHAR2(20), -- 장애, 복구, 재기동 등
    OCCUR_DTM DATE,
    RECOVER_DTM DATE,
    STATUS VARCHAR2(20),
    DESC VARCHAR2(1000)
);
```

---

### 1.4. 설정파일 구조 예시 (YAML)

```yaml
# 전문 레이아웃 예시
messages:
  - id: "TR001"
    fields:
      - name: "userId"
        type: "string"
        length: 20
        mask: true
      - name: "amount"
        type: "number"
        length: 10
  - id: "TR002"
    fields:
      - name: "accountNo"
        type: "string"
        length: 16
        mask: true

# 암호화 정책 예시
encryption:
  - channel: "BANK"
    enabled: true
    algorithm: "AES256"
    keyFile: "/app/keys/bank.key"
  - channel: "CARD"
    enabled: false
```

---

### 1.5. 장애 감지/자동복구/알림 구성 예시

- 서비스 프로세스 헬스 체크:  
  - shell script/agent로 주기적 체크
- 자동 재기동:  
  - 예: `supervisord`, custom shell 활용
- 장애 발생 시  
  - UMS 연동 전문을 통한 SMS 전송  
  - SMTP 서버 통한 이메일 전송
- 알림대상/포맷/빈도는 config 파일로 제어

---

## 2. DR(Disaster Recovery) 시나리오 샘플

### 시나리오 1: DR 모의훈련(주센터 장애 → 백업센터 전환)

1. **사전 준비**
    - DR훈련용 환경설정 파일 생성(예: `application-dr.yml`)
    - DR훈련용 DB, 인증서, 키 등 사전 동기화
    - 운영/DR 환경 분리 배포 및 사전 점검

2. **DR 전환 실행**
    - 주센터 RA 서비스 중지 (모의 장애 발생)
    - 백업센터 환경설정(`application-dr.yml`)로 기동
    - 백업센터에서 채널별 서비스 정상 제공 여부 점검

3. **서비스 정상여부 확인**
    - 주요거래(응답시간, 오류코드, 연계 등) 정상 처리 여부 확인
    - 장애/복구 이벤트 이력 DB 적재 확인

4. **복구 및 원복**
    - 주센터 장애 원인 복구
    - 주센터 정상화 후, DR 환경 서비스 정지 및 원복
    - 이력 및 로그, 장애/복구 이벤트 DB 저장

### 관련 스크립트 예시

```bash
# DR 환경 전환 스크립트 예시 (AIX)
#!/bin/ksh
export APP_HOME=/app/ra
export CONF_FILE=$APP_HOME/config/application-dr.yml
cd $APP_HOME
echo "==== Stop Current RA Service ===="
./stop.sh
echo "==== Start RA Service (DR) ===="
./start.sh --spring.config.location=$CONF_FILE
```

---

## 3. 운영 매뉴얼/체크리스트 예시

- 주요 설정파일(레이아웃, 암호화, 알림 등) 변경/배포 절차
- 장애 발생 시 즉시 조치 매뉴얼
- DR 전환/복구 시나리오별 체크리스트
- UMS, SMTP 연동 설정/테스트 가이드

---

*필요시 설계, 샘플, 매뉴얼 등 추가자료 요청 가능*