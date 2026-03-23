# Woori Card MSA


<img width="2475" height="815" alt="image" src="https://github.com/user-attachments/assets/368ff3b6-eb9a-4483-b4c8-dc60076f1186" />


---
# 시연 가이드

> **Base URL:** `http://172.30.1.33:8080`  (본인의 와이파이 또는 이더넷 ip 환경으로 변경 필요)   
> **공통 헤더:** `Authorization: Bearer {access_token}`

---

## 시연 순서

| # | Method | Endpoint | 설명 |
|---|--------|----------|------|
| 1.0 | `POST` | `/auth/token` | JWT 토큰 발급 |
| 2.0 | `GET` | `/auth/validate` | 토큰 유효성 확인 |
| 3.1.1 | `POST` | `/api/authorizations/request` | 카드 승인 요청 |
| 3.1.2 | `POST` | `/api/authorizations/cancel` | 카드 승인 취소 |
| 3.1.3 | `GET` | `/api/authorizations` | 승인 내역 조회 |
| 3.2.1 | `POST` | `/api/settlements/trigger` | 수동 정산 실행 |
| 3.2.2 | `GET` | `/api/settlements` | 날짜별 정산 조회 |
| 3.2.3 | `GET` | `/api/settlements/merchant/{id}` | 가맹점별 정산 조회 |
| 3.3.1 | `POST` | `/api/billing/monthly` | 청구 생성 |
| 3.3.2 | `GET` | `/api/billing/{cardNumber}` | 청구 내역 조회 |

---

## 1.0 JWT 토큰 발급

```http
POST /auth/token
Content-Type: application/json

{
  "clientId": "vensa",
  "clientSecret": "vensa-secret-2026"
}
```

**응답:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiJ9...",
  "token_type": "Bearer"
}
```

> 발급된 `access_token`을 이후 모든 요청의 `Authorization` 헤더에 사용합니다. (유효기간 1시간)

**Postman Tests 탭 (자동 토큰 저장):**
```javascript
var res = pm.response.json();
pm.environment.set("access_token", res.access_token);
```

> 위 스크립트를 설정하면 이후 요청에서 `{{access_token}}` 변수로 자동 사용 가능

---

## 2.0 토큰 유효성 확인

```http
GET /auth/validate
Authorization: Bearer {access_token}
```

**응답:**
```json
{ "valid": true, "clientId": "vensa" }
```

---

## 3.1.1 카드 승인 요청

```http
POST /api/authorizations/request
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "transactionId": "TXN-007",
  "cardNumber": "6011111111111117",
  "amount": 50000,
  "merchantId": "MERCHANT-001",
  "terminalId": "TERMINAL-001",
  "pin": "1234"
}
```

**응답:**
```json
{
  "transactionId": "TXN-007",
  "approvalNumber": "12345678",
  "responseCode": "00",
  "message": "승인",
  "amount": 50000,
  "approved": true
}
```

> `transactionId`는 매 요청마다 새 값 사용 — `TXN-TEST-001~008`은 seed 데이터로 이미 존재하여 사용 불가 (94: 중복 거래)

### 테스트 카드 (PIN 공통: `1234`)

| 카드번호 | 타입 | 예상 코드 | 설명 |
|----------|------|-----------|------|
| `6011111111111117` | CREDIT | `00` | 정상 승인 (한도 500만) |
| `7011111111111117` | CREDIT | `00` | 정상 승인 (한도 500만) |
| `8011111111111117` | CREDIT | `00` | 정상 승인 (한도 500만) |
| `4111111111111111` | DEBIT  | `00` | 정상 승인 (체크) |
| `3530111333300000` | CREDIT | `61` | 한도 초과 (잔여 50만) |
| `5555555555554444` | DEBIT  | `51` | 잔액 부족 |
| `5105105105105100` | DEBIT  | `14` | 카드 정지 |
| `4012888888881881` | DEBIT  | `54` | 유효기간 만료 |

---

## 3.1.2 카드 승인 취소

```http
POST /api/authorizations/cancel
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "transactionId": "TXN-007",
  "cardNumber": "6011111111111117",
  "cancelReason": "고객 요청"
}
```

**응답:**
```json
{
  "transactionId": "TXN-007",
  "responseCode": "00",
  "message": "취소 성공",
  "cancelled": true
}
```

> CONFIRMED(매출 확정) 상태의 거래는 취소 불가 → `responseCode: 58`

---

## 3.1.3 승인 내역 조회

```http
GET /api/authorizations?date=2026-03-23&status=APPROVED
Authorization: Bearer {access_token}
```

**응답:**
```json
{
  "content": [
    { "transactionId": "TXN-007", "merchantId": "MERCHANT-001", "amount": 50000, "responseCode": "00" }
  ],
  "totalElements": 1,
  "totalPages": 1
}
```

> `date=2026-03-20` 으로 조회하면 seed 데이터 `TXN-TEST-001~008` 목록 확인 가능

---

## 3.2.1 수동 정산 실행

> ⚠️ 정산 조회 전 **반드시 먼저** 실행해야 합니다.

```http
POST /api/settlements/trigger?date=2026-03-23
Authorization: Bearer {access_token}
```

**응답:**
```
2026-03-23 정산 배치 실행 완료
```

> GET으로 호출 시 `405 Method Not Allowed` — 반드시 **POST** 사용

---

## 3.2.2 날짜별 정산 조회

```http
GET /api/settlements?date=2026-03-23
Authorization: Bearer {access_token}
```

**응답:**
```json
[
  {
    "merchantId": "MERCHANT-001",
    "totalCount": 1,
    "totalAmount": 50000,
    "status": "COMPLETED"
  }
]
```

---

## 3.2.3 가맹점별 정산 조회

```http
GET /api/settlements/merchant/MERCHANT-001?from=2026-03-01&to=2026-03-23
Authorization: Bearer {access_token}
```

**응답:**
```json
[
  {
    "settlementDate": "2026-03-23",
    "merchantId": "MERCHANT-001",
    "totalCount": 1,
    "totalAmount": 50000,
    "status": "COMPLETED"
  }
]
```

---

## 3.3.1 청구 생성

> ⚠️ 청구 내역 조회 전 **반드시 먼저** 실행해야 합니다.

```http
POST /api/billing/monthly?cardNumber=6011111111111117&month=2026-03
Authorization: Bearer {access_token}
```

**응답:**
```json
{
  "cardNumberMasked": "6011-****-****-1117",
  "billingMonth": "2026-03",
  "totalAmount": 50000,
  "transactionCount": 1,
  "status": "BILLED",
  "billedAt": "2026-03-23T..."
}
```

> 승인 서비스에서 해당 카드의 해당 월 APPROVED 내역을 조회하여 합산합니다.
> 승인 데이터가 없으면 `totalAmount: 0`, `transactionCount: 0` 으로 생성됩니다.

---

## 3.3.2 청구 내역 조회

```http
GET /api/billing/6011111111111117
Authorization: Bearer {access_token}
```

**응답:**
```json
[
  {
    "cardNumberMasked": "6011-****-****-1117",
    "billingMonth": "2026-03",
    "totalAmount": 50000,
    "transactionCount": 1,
    "status": "BILLED"
  }
]
```

> 3.3.1 청구 생성을 먼저 호출하지 않으면 빈 배열 `[]` 반환

---

## 응답 코드

| 코드 | 설명 |
|------|------|
| `00` | 승인 성공 |
| `14` | 카드 정지 / 카드 정보 없음 |
| `25` | 원거래 없음 (취소 시) |
| `51` | 잔액 부족 (체크카드) |
| `54` | 유효기간 만료 |
| `55` | PIN 오류 |
| `58` | 매출 확정 거래 취소 불가 |
| `61` | 신용 한도 초과 |
| `94` | 중복 거래 |
| `96` | 시스템 오류 |

---

## IP 변경 시 수정 위치

모든 하드코딩 IP는 환경변수로 변경되어 있습니다. **IP가 바뀌면 환경변수만 설정하면 됩니다.**

| 환경변수 | 용도 | 기본값 |
|---------|------|--------|
| `EUREKA_HOST` | Eureka 서버 IP | `172.30.1.33` |
| `HOST_IP` | 각 서비스 자신의 IP (Eureka 등록용) | `172.30.1.33` |

> 게이트웨이의 `HOST_IP` 기본값만 `192.168.1.249`로 다릅니다.

### 수정된 파일 목록

| # | 파일 | 변경 내용 |
|---|------|-----------|
| 1 | `wooricard-gateway/src/main/resources/application.yaml` | `defaultZone: http://${EUREKA_HOST:172.30.1.33}:8761/eureka/` |
| 2 | `wooricard-gateway/src/main/resources/application.yaml` | `ip-address: ${HOST_IP:192.168.1.249}` |
| 3 | `wooricard-config/src/main/resources/application.yaml` | `defaultZone: http://${EUREKA_HOST:172.30.1.33}:8761/eureka/` |
| 4 | `wooricard-approval-service/src/main/resources/application.yml` | `ip-address: ${HOST_IP:172.30.1.33}` |
| 5 | `wooricard-settlement-service/src/main/resources/application.yml` | `ip-address: ${HOST_IP:172.30.1.33}` |
| 6 | `wooricard-billing-service/src/main/resources/application.yml` | `ip-address: ${HOST_IP:172.30.1.33}` |

> `wooricard-eureka`는 `localhost` 고정이므로 변경 불필요.
> `wooricard-config-repo`는 IP 하드코딩 없음.

### 환경변수 설정 방법

**IntelliJ Run Configuration > Environment Variables:**
```
EUREKA_HOST=새IP
HOST_IP=새IP
```

**터미널 직접 실행 시:**
```bash
EUREKA_HOST=새IP HOST_IP=새IP java -jar 서비스명.jar
```

### IP 변경 후 재시작 순서

```
1. wooricard-eureka
2. wooricard-config
3. wooricard-approval-service / wooricard-settlement-service / wooricard-billing-service  (순서 무관)
4. wooricard-gateway
```

---

## 데이터베이스 초기 데이터

승인 서비스는 시작 시 `data.sql`을 `card_authorization_db`에 자동 삽입합니다.
수동으로 넣을 경우 반드시 **`card_authorization_db`** 사용

```sql
USE card_authorization_db;
SELECT COUNT(*) FROM cards;  -- 9건이면 정상
```
