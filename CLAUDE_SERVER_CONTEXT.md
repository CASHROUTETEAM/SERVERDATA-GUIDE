# CLAUDE SERVER CONTEXT — TOTALBOT / CASHROUTE 서버 가이드

> **이 문서는 Claude AI가 서버 작업 시 참조하는 컨텍스트 문서입니다.**
> 사용자: kang in (kangin8697@gmail.com)
> 최종 업데이트: 2026-03-03

---

## 1. 서버 접속 정보

```
Host: 114.202.247.228
User: root
Password: buggy535!!
OS: Ubuntu (Linux)
```

### SSH 접속 헬퍼 스크립트

작업 시 아래 Python 스크립트를 생성하여 사용:

```python
# ssh_cmd.py (30초 타임아웃)
import pexpect, sys
cmd = sys.argv[1]
child = pexpect.spawn(f'ssh -o StrictHostKeyChecking=no root@114.202.247.228 "{cmd}"', timeout=30)
child.expect('password:')
child.sendline('buggy535!!')
child.expect(pexpect.EOF)
print(child.before.decode('utf-8', errors='replace'))
```

```python
# ssh_cmd_long.py (60초 타임아웃 — 대량 쿼리용)
import pexpect, sys
cmd = sys.argv[1]
child = pexpect.spawn(f'ssh -o StrictHostKeyChecking=no root@114.202.247.228 "{cmd}"', timeout=60)
child.expect('password:')
child.sendline('buggy535!!')
child.expect(pexpect.EOF)
print(child.before.decode('utf-8', errors='replace'))
```

```python
# scp_upload.py (로컬 → 서버 파일 전송)
import pexpect, sys
local, remote = sys.argv[1], sys.argv[2]
child = pexpect.spawn(f'scp -o StrictHostKeyChecking=no {local} root@114.202.247.228:{remote}', timeout=30)
child.expect('password:')
child.sendline('buggy535!!')
child.expect(pexpect.EOF)
print(child.before.decode('utf-8', errors='replace'))
print("Upload done")
```

### SSH 명령어 실행 패턴

**중요: sqlite3 쿼리는 직접 SSH로 실행 시 따옴표/괄호 이스케이핑 문제 발생**
→ 반드시 `.sh` 스크립트 파일을 생성 → SCP 업로드 → `bash /tmp/스크립트.sh`로 실행

```bash
# 올바른 패턴
# 1. 로컬에서 .sh 파일 생성
# 2. scp_upload.py로 /tmp/에 업로드
# 3. ssh_cmd.py "bash /tmp/파일.sh" 로 실행
```

---

## 2. 서버 아키텍처 개요

```
┌─────────────────────────────────────────────────────┐
│                   서버 (114.202.247.228)                │
│                                                         │
│  ┌──────────────────┐    ┌──────────────────────┐      │
│  │  Flask (Python)   │    │  Node.js (Express)    │      │
│  │  Gunicorn         │    │  PM2 관리              │      │
│  │  Port: 5000       │    │  Port: 3000            │      │
│  │                    │    │                        │      │
│  │  담당:             │    │  담당:                  │      │
│  │  - 물류 관리       │    │  - 상품 수집            │      │
│  │  - 입출고          │    │  - 견적서               │      │
│  │  - 반출            │    │  - 누끼 (배경제거)      │      │
│  │  - 재고 관리       │    │  - AI 이미지            │      │
│  │  - 유저/포인트     │    │  - 크롤링               │      │
│  │  - 관리자 대시보드 │    │  - JWT 인증             │      │
│  │  - 회송            │    │  - 챗봇                 │      │
│  │  - 청구/급여       │    │  - 쿠팡/네이버 연동     │      │
│  │                    │    │  - 정산                  │      │
│  └────────┬───────────┘    └────────┬───────────────┘      │
│           │                          │                      │
│  ┌────────▼───────────┐    ┌────────▼───────────────┐      │
│  │  user_data.db       │    │  database.sqlite        │      │
│  │  (Flask DB)         │    │  (Node DB)              │      │
│  │  345MB / SQLite     │    │  317MB / SQLite         │      │
│  └─────────────────────┘    └─────────────────────────┘      │
└──────────────────────────────────────────────────────────────┘
```

### 프로세스 관리

```bash
# Flask 재시작
sudo systemctl restart gunicorn

# Node.js 재시작
cd /var/www/totalbotserver/totalbot_node && pm2 restart all

# 상태 확인
pm2 status
sudo systemctl status gunicorn
```

---

## 3. 파일 경로 맵

```
/var/www/totalbotserver/
├── app.py                          # Flask 메인 (3만줄+, 매우 큼)
├── user_data.db                    # Flask DB (345MB) — 물류/유저/포인트
├── version.json                    # 데스크톱 앱 버전 관리
├── updates/                        # 앱 업데이트 파일 배포 폴더
├── claude_work/                    # Claude 작업 스크립트 저장 폴더
│
├── totalbot_node/
│   └── server/
│       ├── server.js               # Node 메인 엔트리포인트
│       ├── database.sqlite         # Node DB (317MB) — 상품수집/인증
│       ├── data/
│       │   └── users.json          # 유저 목록 (29명, 마이그레이션 원본)
│       ├── middleware/
│       │   └── auth.js             # JWT 검증 미들웨어
│       ├── routes/
│       │   ├── auth.js             # 로그인/회원가입 (★ 수정됨: Flask DB 등급 동기화)
│       │   ├── magicEraser.js      # 누끼 (배경제거)
│       │   ├── gemini.js           # AI 이미지 생성
│       │   ├── quote.js            # 견적서
│       │   ├── orders.js           # 주문 관리
│       │   ├── order.js            # 주문 상세
│       │   ├── crawl.js            # 크롤링
│       │   ├── coupang.js          # 쿠팡 연동
│       │   ├── upload.js           # 파일 업로드
│       │   ├── excel.js            # 엑셀 처리
│       │   ├── chatbot.js          # 챗봇
│       │   ├── points.js           # 포인트 (Node 측)
│       │   ├── settlement.js       # 정산
│       │   ├── priceHistory.js     # 가격 이력
│       │   └── sqlite-viewer.js    # DB 뷰어
│       └── utils/
│           └── db.js               # DB 유틸 (getDb(), SQLITE_USER_IDS)
│
├── flutter_inventory_scanner/      # 모바일 앱 (Flutter)
│   └── lib/
│       └── main.dart               # 재고스캐너 앱 메인
```

---

## 4. 데이터베이스 상세

### 4-1. Flask DB (user_data.db)

**주요 테이블:**

```sql
-- ★ users (573명, 핵심 유저 테이블)
CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    username VARCHAR(50) NOT NULL,       -- 로그인 ID (전화번호 또는 아이디)
    password VARCHAR(100) NOT NULL,
    name VARCHAR(50),
    phone VARCHAR(30),
    email VARCHAR(120),
    credits INTEGER DEFAULT 0,           -- ★ 포인트 잔액
    grade VARCHAR(20) DEFAULT 'basic',   -- ★ premium/student/basic
    is_admin BOOLEAN DEFAULT 0,
    is_subscribed BOOLEAN DEFAULT 0,     -- 구독 여부
    image_nukki_limit INTEGER DEFAULT 50,  -- 누끼 한도 (-1=무제한)
    image_nukki_used INTEGER DEFAULT 0,
    ai_image_limit INTEGER DEFAULT 20,     -- AI이미지 한도
    ai_image_used INTEGER DEFAULT 0,
    premium_days_remaining INTEGER DEFAULT 0,
    subscription_start_date DATETIME,
    subscription_end_date DATETIME,
    created_at DATETIME
);

-- 등급 분포: premium=485, student=85, basic=3
-- ★ student/premium → image_nukki_limit=-1(무제한), ai_image_limit=100
-- ★ -1은 무제한 의미 (app.py에서 limit != -1 체크)

-- ★ point_history (포인트 변동 이력)
CREATE TABLE point_history (
    id INTEGER PRIMARY KEY,
    user_id INTEGER NOT NULL,
    category VARCHAR(50) NOT NULL,       -- '출고','입고','반출','박스비','회송' 등
    amount INTEGER NOT NULL,             -- 양수=충전, 음수=차감
    balance_after INTEGER NOT NULL,
    description TEXT,
    reference_id VARCHAR(100),           -- 관련 작업 참조번호
    created_at DATETIME,
    processed_by INTEGER                 -- 처리 작업자 ID
);

-- 주요 카테고리: 출고(40310), 입고(3009), 반출(2333), 박스비(1700),
--              회송(1343), 택배비(1081), 쉽먼트부착비(1072) ...

-- ★ inventory_data (재고)
CREATE TABLE inventory_data (
    id INTEGER PRIMARY KEY,
    username VARCHAR(100),
    sku VARCHAR(100),
    barcode VARCHAR(100),
    product_name VARCHAR(500),
    brand VARCHAR(200),
    location VARCHAR(200),           -- 예: 'WH/Stock/A4-510'
    quantity INTEGER,
    reserved_quantity INTEGER,
    created_at DATETIME,
    updated_at DATETIME,
    pending_amount INTEGER DEFAULT 0
);

-- ★ in_out_list (입출고 기록)
CREATE TABLE in_out_list (
    id INTEGER PRIMARY KEY,
    move_id INTEGER,
    username VARCHAR(100),
    date DATETIME,
    move_type VARCHAR(50),           -- '입고','출고','반출','회송','미청구 입고'
    reference VARCHAR(200),
    shipment_number VARCHAR(100),
    center VARCHAR(200),
    departure_location VARCHAR(500),
    arrival_location VARCHAR(500),
    sku VARCHAR(100),
    item VARCHAR(500),
    barcode VARCHAR(100),
    item_brand VARCHAR(200),
    quantity FLOAT,
    created_at DATETIME
);

-- ★ export_tasks (반출 작업)
CREATE TABLE export_tasks (
    id INTEGER PRIMARY KEY,
    reference VARCHAR(50) NOT NULL UNIQUE,  -- 'WH/EXPORT/00076'
    brand_name VARCHAR(200) NOT NULL,       -- '0227 반출1'
    reason VARCHAR(500),                     -- '6개월이상 미입고'
    status VARCHAR(20),                      -- 'pending','in_progress','completed'
    total_products INTEGER,
    total_quantity INTEGER,
    scanned_quantity INTEGER,
    created_by INTEGER,
    completed_by INTEGER,
    created_at DATETIME,
    updated_at DATETIME,
    completed_at DATETIME,
    is_partial BOOLEAN DEFAULT 0
);

-- ★ export_task_items (반출 품목)
CREATE TABLE export_task_items (
    id INTEGER PRIMARY KEY,
    task_id INTEGER NOT NULL,
    barcode VARCHAR(100) NOT NULL,
    sku VARCHAR(100),
    product_name VARCHAR(500),
    location VARCHAR(200),
    username VARCHAR(100),
    user_id INTEGER,
    expected_quantity INTEGER,
    scanned_quantity INTEGER,
    status VARCHAR(20),                  -- 'pending','partial','completed'
    scanned_at DATETIME,
    scanned_by INTEGER
);

-- return_records (회송 기록, 935건)
-- return_reasons (회송 사유, 15종)

-- 기타 테이블들:
-- workers (작업자), attendances (출근), payroll_records (급여)
-- delivery_cost_records, other_cost_records
-- order_lists, picking_tasks, picking_items
-- outbound_shipments, outbound_shipment_items
-- box_shipping_records, tote_boxes
-- shipment_data, route_style_shipments
-- point_charge_requests, point_refund_requests
-- user_rate_settings, default_rate_settings
-- default_salary_settings, salary_settings
```

### 4-2. Node DB (database.sqlite)

```sql
-- users (23명, Node 측 인증용)
-- grade 필드는 Flask DB와 동기화됨 (auth.js에서 로그인 시 자동 동기화)
-- SQLITE_USER_IDS: [1,3,5,6,7,8,9,10,11,12,13,14,15,16,17,19,20,21,22,24,25,26,27]

-- products (상품 수집 데이터)
-- 3,295개 상품, 19명 유저, 3개월간 수집
-- 피크: 하루 137개

-- users.json (29명, 마이그레이션 원본)
-- migrate_to_sqlite_efficient.js로 database.sqlite에 마이그레이션됨
```

---

## 5. 핵심 비즈니스 로직

### 5-1. 등급 시스템 & 권한

```
grade 종류: premium, student, basic
- premium/student → 견적서 가능, 누끼 무제한, AI이미지 100개
- basic → 견적서 불가, 누끼 20개 (구독자는 500개), AI이미지 5개

★ 중요: grade는 Flask DB가 원본 (Flask 관리자 대시보드에서 변경)
   Node DB는 로그인 시 auth.js에서 Flask DB 조회 후 자동 동기화
```

### 5-2. 포인트 시스템

```
- 유저 잔액: users.credits
- 출고 시 차감: 건당 단가 × 수량 (기본 100원, 유저별 커스텀 가능)
- 반출 시 차감: 건당 단가 × 수량 (user_rate_settings.shipping_cost)
- 입고 시 차감: 건당 과금
- 포인트 충전: 관리자가 직접 충전
- 마이너스 허용 (minus_point_start_date 기록)
```

### 5-3. 반출 프로세스

```
1. 엑셀 업로드 → export_tasks + export_task_items 생성 (status: pending)
2. 작업자 바코드 스캔 → (status: in_progress)
   - inventory_data 재고 차감
   - in_out_list에 move_type='반출' 기록
   - point_history에 포인트 차감 (category='반출')
   - users.credits 차감
3. 전체 스캔 완료 → (status: completed)

★ 스캔 안 된 항목: 재고/포인트 변동 없음
★ 부족 확정 (confirm_shortage): 재고 처리 없이 status만 completed로 변경
```

### 5-4. 이미지 한도 체크 (app.py)

```python
# 누끼 한도 조회 로직
'image_nukki_limit': user.image_nukki_limit if user.image_nukki_limit is not None
                     else (500 if user.is_subscribed else 20)
'ai_image_limit': user.ai_image_limit if user.ai_image_limit is not None
                  else (100 if user.is_subscribed else 5)

# 무제한 체크 (-1이면 스킵)
if limit != -1 and current_used + count > limit:
    return error "한도 초과"
```

### 5-5. 위치 체계

```
WH/Stock/{구역}{번호}-{선반}
예: WH/Stock/A4-510

구역: A1~A9, B1~B3, C1~C2, D1~D4, E1~E3, F1~F3, P, R, Z
(P/R/Z는 2026-03 추가, R=구 RF, Z=구 TE)
```

---

## 6. 수정 이력

### 6-1. auth.js — Flask DB 등급 동기화 추가

```
파일: /var/www/totalbotserver/totalbot_node/server/routes/auth.js
백업: auth.js.backup_20260227

변경 내용:
- getGradeFromFlaskDb() 함수 추가: Flask user_data.db에서 grade 조회
- 로그인 시 Flask DB grade와 Node DB grade 비교
- 불일치 시 Flask DB 값으로 Node DB 자동 업데이트
- JWT 토큰에 Flask DB grade 반영
```

### 6-2. app.py — 이미지 한도 무제한 처리

```
파일: /var/www/totalbotserver/app.py
백업: app.py.backup_limits_20260227

변경 내용:
- image_nukki_limit = -1 이면 무제한 (한도 체크 스킵)
- 한도 초과 필터 쿼리에 != -1 조건 추가
- 라인 ~33746, ~33759: limit != -1 체크 추가
- 라인 ~33479, ~33548: 필터 쿼리에 조건 추가
```

### 6-3. user_data.db — 등급별 이미지 한도 업데이트

```
UPDATE users SET image_nukki_limit = -1, ai_image_limit = 100
WHERE grade IN ('student', 'premium');
-- 570명 적용 (premium 485 + student 85)

-- 01043865800 (김영자): basic→premium으로 변경 + 한도 적용
```

### 6-4. 반출 관련 작업 (0227 반출1~5)

```
-- 포인트 환불: 10명, 총 20,400원 (category='반출', description에 ROLLBACK_20260227)
-- in_out_list 반출 기록 47건 삭제 (재고는 복원 안 함)
-- export_tasks/items는 유지 (삭제 후 백업에서 복원)
-- 반출4(id=79), 반출5(id=80): completed → in_progress로 변경 후 재진행
```

### 6-5. 2026-02-28 추가 변경

```
-- 반출 0227 전체 최종 완료:
   반출1(76): 245개중 243개 스캔 completed
   반출2(77): 190개중 190개 스캔 completed (2026-02-28 12:29)
   반출3(78): 196개중 195개 스캔 completed
   반출4(79): 156개중 156개 스캔 completed (2026-02-28 12:44)
   반출5(80): 174개중 172개 스캔 completed

-- 무료입고 대량 처리: WH/입고/FREE/00373~00381 (약 9건 배치)
-- 새 백업: user_data.db.backup_export_rollback_20260228_041102
```

### 6-6. 2026-03-03 추가 변경

```
-- version.json에 신규 앱 항목 추가:
   balzubotversion: 1.1.9 → http://114.202.247.228/updates/balzubot_1.1.9.zip
   zangsanbotversion: 1.0.2 → http://114.202.247.228/updates/zangsanbot_1.0.2.zip
   imagebot_version: 1.0.0 → http://114.202.247.228/updates/imagebot_1.0.0.zip

-- 재고 구역 변경: RF→R, TE→Z로 명칭 변경, P 구역 신규 추가
   현재 구역: A/B/C/D/E/F/P/R/Z

-- happymine(김호연) 포인트 0P로 조정 (관리자 직접 수정, -17,300P)
-- fktpstm83(배소연) 150,000P 충전 (REQ-316, 에드(ADD) 입금)
-- pm2 재시작 횟수: 59회 (2026-02-26 기준 46회에서 증가)
```

---

## 7. 주의사항

### SQL 실행 시

```
1. sqlite3 쿼리는 반드시 .sh 스크립트 → SCP → bash 실행 패턴 사용
2. 한글 컬럼 별칭(alias)은 SELECT 내부에서만 사용, WHERE/GROUP BY에서 사용 불가
3. 대량 UPDATE 전 반드시 백업: cp DB경로 DB경로.backup_날짜시간
4. user_data.db는 345MB — 전체 다운로드 비권장
```

### app.py 수정 시

```
1. 3만줄 이상 — 전체 읽기 불가, sed -n '시작,끝p' 사용
2. 수정 후 반드시: sudo systemctl restart gunicorn
3. 백업: cp app.py app.py.backup_설명_날짜
```

### Node 수정 시

```
1. 수정 후: cd /var/www/totalbotserver/totalbot_node && pm2 restart all
2. 로그 확인: pm2 logs
```

### 이중 DB 주의

```
★ Flask DB (user_data.db) = 물류/포인트/유저관리의 원본
★ Node DB (database.sqlite) = 상품수집/인증용
★ grade는 Flask DB가 마스터, Node DB는 로그인 시 동기화
★ 포인트(credits)는 Flask DB에만 존재
```

---

## 8. 모바일 앱 (재고스캐너)

```
위치: /var/www/totalbotserver/flutter_inventory_scanner/
프레임워크: Flutter (Dart)
기능: Deep Link (inventoryscanner://scan)

API 엔드포인트 (Flask):
- POST /api/inventory_check/start      — 재고조사 시작
- POST /api/inventory_check/scan       — 바코드 스캔
- GET  /api/inventory_check/status/<id> — 조사 상태
- POST /api/inventory_check/complete    — 조사 완료
- GET  /api/inventory_check/history     — 이력 조회
- GET  /api/inventory_check/detail/<id> — 상세 조회

버전 관리: /var/www/totalbotserver/version.json
업데이트: /var/www/totalbotserver/updates/ 폴더
```

---

## 9. 데스크톱 앱 버전 관리

```json
// /var/www/totalbotserver/version.json (2026-03-03 기준)
{
    "version": "1.1.6",
    "update_url": "http://114.202.247.228/updates/cashbot_1.1.6.zip",

    "balzubotversion": "1.1.9",
    "balzubot_update_url": "http://114.202.247.228/updates/balzubot_1.1.9.zip",

    "zangsanbotversion": "1.0.2",
    "zangsanbot_update_url": "http://114.202.247.228/updates/zangsanbot_1.0.2.zip",

    "imagebot_version": "1.0.0",
    "imagebot_update_url": "http://114.202.247.228/updates/imagebot_1.0.0.zip",

    "totalbot_version": "1.0.60",
    "totalbot_update_url": "http://114.202.247.228/updates/totalbot_1.0.49.zip"
}
```

---

## 10. 자주 쓰는 조회 쿼리

```sql
-- 유저 정보 조회
SELECT id, username, name, grade, is_subscribed, credits,
       image_nukki_limit, ai_image_limit
FROM users WHERE username = '대상유저';

-- 포인트 이력 조회
SELECT * FROM point_history WHERE user_id = 유저ID ORDER BY id DESC LIMIT 20;

-- 재고 조회 (바코드 기준)
SELECT * FROM inventory_data WHERE barcode = '바코드' AND quantity > 0;

-- 반출 작업 현황
SELECT id, reference, brand_name, status, total_quantity, scanned_quantity
FROM export_tasks ORDER BY id DESC LIMIT 10;

-- 6개월 미출고 SKU 조회
SELECT inv.username, inv.barcode, inv.product_name, inv.location, inv.quantity,
       COALESCE(lo.last_out, '출고기록없음') as last_out
FROM inventory_data inv
LEFT JOIN (
  SELECT barcode, username, MAX(date) as last_out
  FROM in_out_list WHERE move_type IN ('출고', '반출')
  GROUP BY barcode, username
) lo ON lo.barcode = inv.barcode AND lo.username = inv.username
WHERE inv.quantity > 0
  AND (lo.last_out IS NULL OR lo.last_out < date('now', '-6 months'));

-- 등급 분포
SELECT grade, is_subscribed, COUNT(*) FROM users GROUP BY grade, is_subscribed;

-- 포인트 카테고리별 합계
SELECT category, COUNT(*), SUM(amount) FROM point_history GROUP BY category;
```

---

## 11. 백업 목록 (서버 내)

```
/var/www/totalbotserver/user_data.db.backup_20260224_161449          (312MB)
/var/www/totalbotserver/user_data.db.backup_export_rollback_20260227173159  (345MB)
/var/www/totalbotserver/user_data.db.backup_export_rollback_20260228_041102 (346MB) ← 최신
/var/www/totalbotserver/app.py.backup_limits_20260227
/var/www/totalbotserver/totalbot_node/server/routes/auth.js.backup_20260227
/root/backup_20260226/app.py.bak, app.py.bak2, app.py.bak3
```

---

## 12. 미완료 작업 (TODO)

```
[ ] 관리자 대시보드 + Worker 서버 통합 (개발자 이미지 마이그레이션 완료 후)
[ ] 포인트 관리 + 포인트 신청 + 포인트 분석 통합
[ ] 가이드유 회원 통합
[ ] 스캐너 앱 version.json 항목 추가
[ ] 반출 0227 반출1~5 이후 나머지 2,070개 미출고 SKU 반출 처리
[x] 반출 0227 반출1~5 포인트 환불 및 재진행 (완료: 2026-02-28)
[x] balzubot/zangsanbot/imagebot version.json 추가 (완료: 2026-03-03)
```
