# CASHROUTE/TOTALBOT 서버 작업 컨텍스트
> 최종 업데이트: 2026-03-05

---

## 접속 정보
```
Host: 114.202.247.228
User: root
Password: buggy535!!
서비스명: gunicorn-totalbot.service (주의 — gunicorn.service 아님)
```

---

## SSH 헬퍼 스크립트 (매 세션마다 생성)

```python
# ssh_cmd.py (30초)
import pexpect, sys
child = pexpect.spawn(f'ssh -o StrictHostKeyChecking=no root@114.202.247.228 "{sys.argv[1]}"', timeout=30)
child.expect('password:'); child.sendline('buggy535!!'); child.expect(pexpect.EOF)
print(child.before.decode('utf-8', errors='replace'))

# ssh_cmd_long.py (60초 — 대량 쿼리용)
import pexpect, sys
child = pexpect.spawn(f'ssh -o StrictHostKeyChecking=no root@114.202.247.228 "{sys.argv[1]}"', timeout=60)
child.expect('password:'); child.sendline('buggy535!!'); child.expect(pexpect.EOF)
print(child.before.decode('utf-8', errors='replace'))

# scp_upload.py (로컬→서버)
import pexpect, sys
child = pexpect.spawn(f'scp -o StrictHostKeyChecking=no {sys.argv[1]} root@114.202.247.228:{sys.argv[2]}', timeout=30)
child.expect('password:'); child.sendline('buggy535!!'); child.expect(pexpect.EOF)
print("Upload done")
```

**sqlite3 쿼리 실행 규칙:** 직접 SSH 실행 시 이스케이핑 오류 발생. 반드시 `.sh` 파일 작성 → scp 업로드 → `bash /tmp/스크립트.sh` 패턴 사용. 한글 컬럼 별칭 사용 불가(영문만).

---

## 서비스 재시작

```bash
# Flask (★ 반드시 gunicorn-totalbot.service 사용)
sudo systemctl restart gunicorn-totalbot.service
sudo systemctl status gunicorn-totalbot.service

# Node.js
cd /var/www/totalbotserver/totalbot_node && pm2 restart all
pm2 status
```

---

## 핵심 파일 경로

```
/var/www/totalbotserver/
├── app.py                    # Flask 메인 (3만줄+) — sed -n '시작,끝p' 로 부분 조회
├── user_data.db              # Flask DB (345MB) — 물류/유저/포인트 원본
├── totalbot_node/server/
│   ├── database.sqlite       # Node DB (317MB) — 상품수집/인증
│   └── routes/auth.js        # 로그인/회원가입 (Flask DB 등급 동기화)
```

---

## 인증 흐름 (중요)

- **프로그램(데스크톱) 로그인** → Flask `/login` 사용 → Flask DB 유저 전부 로그인 가능
- **Node auth.js** → 웹 기능 전용, 프로그램 로그인과 무관
- **Node DB** → 프로그램 `/register` API로 직접 가입한 23명만 존재
- **grade** → Flask DB가 마스터, Node DB는 로그인 시 자동 동기화

---

## TTL 캐시 설정 (app.py 1252~1255행)

```python
product_info_cache = TTLCache(default_ttl=600, max_size=500)   # 제품 기본정보: 10분
stock_qty_cache    = TTLCache(default_ttl=30,  max_size=500)   # 재고 수량: 30초
inout_history_cache = TTLCache(default_ttl=300, max_size=200)  # 입출고 이력: 5분
```

**캐시 무효화 위치 (2026-03-05 수정 후 총 7곳):**

| 위치(행) | 작업 | 무효화 대상 |
|---|---|---|
| ~1005 | Odoo 출고 (기존) | stock + inout + product |
| ~10022 | 재고조사 수량 수정 | stock + product만 (inout 제외) |
| ~14999 | 피킹 완료 | stock + inout + product |
| ~26221 | 입고 처리 | stock + inout + product |
| ~26707 | 회송 입고 | stock + inout + product |
| ~26912 | 회송 무료입고 | stock + inout + product |
| ~27928 | 반출 스캔 | stock + inout + product |

무효화 코드 패턴:
```python
stock_qty_cache.invalidate()
inout_history_cache.invalidate()
product_info_cache.invalidate("inv_base_")
product_info_cache.invalidate("prod_base_")
```

---

## 발주처리(발주 자동화) 시스템

### API 호출 순서 (프로그램 → 서버)
1. `GET /api/shipment_data/grouped_summary?username=...` — 납품 데이터 조회
2. `GET /api/product_list/by_username?username=...` — 재고/발주필요여부 조회
3. 프로그램 내부 계산 (서버에서 볼 수 없음)
4. `POST /api/shipment_data/bulk_import` — 최종 발주 데이터 저장

### shipment_data 주요 컬럼
```sql
shipment_number  -- 쉽먼트 번호
barcode          -- 바코드
quantity         -- 필요 수량 (센터에 보내야 할 수량)
shipped_quantity -- 실제 출고된 수량
center           -- 센터명 (인천45, 천안, 안산3 등)
ship_start_date  -- 입고예정일 (★ +14일이 거래처 수령 마감)
ship_end_date    -- 발송 마감일
```

### 발주 유효기한 계산
- 거래처 수령 가능 기간: **ship_start_date + 14일**
- 기한 만료 쉽먼트는 발주 계산에서 제외
- SQL: `date(ship_start_date, '+14 days') >= date('now')`

### product_list 주요 컬럼
```sql
total_stock      -- 총 재고
reserved_stock   -- 예약(미출고) 수량 합계 — ★ 전 센터 합산값 (센터 구분 없음)
available_stock  -- 가용 재고
coming_stock     -- 입고 예정(발주됨) 수량
needs_double_order -- 초도 2배 주문 필요 여부
```

### 알려진 발주 오계산 패턴 (2026-03-05 확인)
- **증상**: 인천36에 1개만 필요한데 2개 주문, 안산3에 4개 필요한데 3개 주문
- **원인 추정**: 프로그램이 센터별 개별 계산이 아닌 "전체 미출고 합계 - coming_stock" 누적 방식으로 계산
- **서버에서 역추적 가능한 범위**: bulk_import 입력값(shipment_data) vs 출력값(주문수량) 비교까지만 가능. 실제 계산 로직은 프로그램(데스크톱 앱) 소스코드에 있음

---

## 초도 중첩 발송 (ZA 2배 주문) 시스템

### 개요
한번도 출고된 적 없는 신규 바코드의 경우, 발송 가능 기간 내 동일 바코드 쉽먼트가 2개 이상 있어야 발송하는 기능. 조건 충족 시 원 주문량만큼 ZA 쉽먼트로 추가 주문.

### 서버 API (모두 구현 완료 — 2026-03-05 확인)

```
GET/POST /api/user/require_duplicate_barcode  — 기능 ON/OFF 설정 (20275행~)
GET/POST /api/user/max_initial_order_wait     — 초도 대기 차수 설정
GET      /api/product_list/by_username        — needs_double_order 필드 포함 반환 (25450행~)
POST     /api/in_out_list/check_barcodes      — 바코드 출고이력 확인 (25032행~, username 무관)
```

### needs_double_order 판단 로직 (app.py 25462행~)
```python
if valid_count == -1:      → False (이미 발송됨)
elif exceeded_max:         → False (최대 대기 차수 초과)
elif valid_count < 2:      → True  (초도 대기 — 겹치는 쉽먼트 부족)
else:                      → True  (중첩 준비 완료)
```

### check_barcodes API (app.py 25032행~)
- in_out_list + shipment_data(shipped_quantity>0) 양쪽 모두 확인
- 500개씩 청크 처리
- username 필터 없음 — 어떤 계정에서 출고했든 이력 있으면 반환

### ZA 쉽먼트 번호 규칙
- 사업자번호 있으면: `{사업자번호}ZA` (예: 3531402947ZA)
- 브랜드명 있으면: `{브랜드명}ZA`
- 둘 다 없으면: `{원래쉽먼트번호}ZA`

---

## 유저 관련

### users 테이블 추가 컬럼 (스키마 기준)
```sql
require_duplicate_barcode BOOLEAN  -- 초도 중첩 발송 ON/OFF
max_initial_order_wait INTEGER     -- 초도 대기 차수
```

### 주요 유저
- xter7570 (id=10, 이명수, student, 1,150,850P) — 구 xter4140 데이터 통합됨
- admin (임준혁): officialcashroute@gmail.com

---

## 백업 목록 (전체)

```
/var/www/totalbotserver/
├── app.py.backup_cache_invalidate_20260305     ← 캐시 무효화 수정 전
├── app.py.backup_limits_20260227
├── user_data.db.backup_xter_merge_20260305     ← xter4140→xter7570 통합 전
├── user_data.db.backup_export_rollback_20260227173159
├── totalbot_node/server/routes/auth.js.backup_20260227

/root/backup_20260226/
├── app.py.bak      (최초 원본)
├── app.py.bak2     (인덱스+reference 수정 후)
├── app.py.bak3     (캐싱 적용 전)
└── picking_process.html.bak
```

---

## 미완료 작업 (TODO)

```
[ ] 발주 오계산 근본 수정 — 프로그램 소스코드 필요 (깃허브 업로드 후 분석)
    → generate_orders() 함수의 need 값 계산 로직 확인 필요
    → 현상: 센터별 개별 계산 아닌 전체 합산 방식으로 추정
[ ] 관리자 대시보드 + Worker 서버 통합
[ ] 포인트 관리 + 포인트 신청 + 포인트 분석 통합
[ ] 가이드유 회원 통합
[ ] 스캐너 앱 version.json 항목 추가
[ ] 반출 0227 반출1~5 이후 나머지 2,070개 미출고 SKU 반출 처리
[ ] SQL 인젝션 가짜 계정 79개 정리 + 보안 강화 (workers 테이블, fnfOzvSR 패턴)
[ ] SSH 포트 변경 또는 키 인증 전용 전환
[ ] lecture-frontend Vite 개발서버 → 프로덕션 빌드 전환
[ ] RAM 업그레이드 (최소 8GB 권장)
```

---

## 자주 쓰는 쿼리

```sql
-- 유저 정보
SELECT id, username, name, grade, credits FROM users WHERE username = '대상유저';

-- 재고 조회
SELECT * FROM inventory_data WHERE barcode = '바코드' AND quantity > 0;

-- 특정 바코드 쉽먼트 전체 (유효기한 포함)
SELECT shipment_number, quantity, shipped_quantity, center,
       ship_start_date, date(ship_start_date, '+14 days') as deadline
FROM shipment_data
WHERE barcode = '바코드' AND username = '유저'
ORDER BY ship_start_date;

-- 유효한 미출고 쉽먼트 (기한 내)
SELECT * FROM shipment_data
WHERE username = '유저'
  AND (quantity - COALESCE(shipped_quantity,0)) > 0
  AND date(ship_start_date, '+14 days') >= date('now')
ORDER BY ship_start_date;

-- product_list 발주 현황
SELECT barcode, total_stock, reserved_stock, available_stock, coming_stock
FROM product_list WHERE username = '유저' AND barcode = '바코드';

-- in_out_list 이력
SELECT move_type, quantity, shipment_number, center, date, created_at
FROM in_out_list WHERE barcode = '바코드' ORDER BY created_at DESC LIMIT 20;
```
