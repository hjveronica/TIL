# DDI Checker — 시스템 흐름 문서
> 최종 업데이트: 2026-05-04  
> 이 문서는 코드에서 직접 추출되었습니다. (추정) 표시는 추론 항목, **미구현** 표시는 코드에 없는 항목입니다.

---

## 1. 시스템 구성 요소

| 레이어 | 기술 스택 | 파일/리소스 |
|--------|----------|------------|
| 프론트엔드 | React 18 + TypeScript + Vite + Tailwind CSS | `ddi-ui/src/` |
| API Gateway | AWS API Gateway REST (prod 스테이지) | `https://d15736fmnl.execute-api.ap-northeast-2.amazonaws.com/prod` |
| Lambda | Python 3.12 | `ddi-check/handler.py` → `lambda_handler()` |
| DB (메인) | PostgreSQL (AWS RDS) | `drug_mapping`, `ddi_pairs`, `user_profile`, `ddi_pregnancy_flags`, `ddi_age_flags`, `ddi_logs` |
| 복용약 저장소 | DynamoDB | `say2-3team-ddi-drugs` (PK: user_id, SK: drug_id) |
| DDI 캐시 | DynamoDB (Phase 1) / Redis (Phase 2 미구현) | `say2-3team-ddi-pairs` (PK: user_id, SK: pair_key) |
| 간접 DDI | Amazon Neptune (효소 경로 그래프) | `core/neptune_client.py` — 현재 연결 불안정, NEPTUNE_UNAVAILABLE warning 처리 |
| 인증 | AWS Secrets Manager | `ddi/api/key` (API Key), `ddi/rds/credentials` (PostgreSQL) |
| 미매핑 처리 | Amazon SQS | `SQS_UNMAPPED_URL` env var → `core/drug_mapper._publish_unmapped()` |
| 뷰 어댑터 | 프론트엔드 레이어 | `ddi-ui/src/lib/ddiAdapter.ts` |

---

## 2. 전체 요청 흐름

### 정상 경로 (Happy Path)

```
사용자 입력 → API 호출 → Lambda 처리 → 응답 렌더링
```

1. **사용자가 환자ID + 신규약물명 입력 후 제출**  
   → `DDICheckPage.tsx` `handleSubmit()` 호출  
   → `checkDDI(req)` via `api/ddi.ts`

2. **POST /ddi-check 요청 전송**  
   Headers: `x-api-key`, `Content-Type: application/json`  
   Body: `{ user_id, new_drug_name, view_type, include_indirect }`

3. **API Gateway → Lambda 호출**  
   → `handler.py` `lambda_handler(event, context)`

4. **API Key 검증** (`handler._verify_api_key()`)  
   → Secrets Manager `ddi/api/key` 조회 (cold start 1회 캐시)  
   → 불일치 시: `401 UNAUTHORIZED` 즉시 반환

5. **Request 파싱** → `DDICheckRequest` Pydantic 모델  
   → 실패 시: `400 INVALID_INPUT`

6. **`pair_engine.run(request)` 호출**

7. **약물명 해소** (`drug_mapper.map_drug(new_drug_name)`)  
   6단계 Resolution 순서:
   ```
   D001 형식 → ingredient_name_kr ILIKE (한글만) → product_name_kr ILIKE (한글만)
   → alias[] / ingredient_name_en ILIKE → MFDS API → SQS 발행
   ```
   → 실패 시: `raise ValueError("DRUG_NOT_FOUND")` → `404`

8. **복용약 목록 조회** (`cache.get_user_medications(user_id)`)  
   → DynamoDB `say2-3team-ddi-drugs` query (end_date 없는 항목만)  
   → 빈 목록 시: `DDICheckResponse(total_pairs=0)` 즉시 반환 (HTTP 200)  
   → DynamoDB 실패 시: `raise ValueError("USER_NOT_FOUND")` → `404`

9. **캐시 분리** (`pair_engine._split_cache()`)  
   → 각 (new_drug, current_drug) 쌍을 `make_pair_key(a, b)` = 알파벳 정렬 `#` 연결  
   → DynamoDB 또는 Redis 조회 → hits / misses 분리

10. **병렬 I/O** (`pair_engine._run_parallel_io()` — ThreadPoolExecutor 2개)  
    - **Worker A**: `rds_client.query_ddi_pairs(cache_misses)` + `query_supplement_interactions()`  
      → PostgreSQL `ddi_pairs`, `drug_supplement_interactions` 배치 조회  
    - **Worker B**: `neptune_client.get_indirect_ddi_cached(new_drug_id, current_drug_ids)` (include_indirect=True 시)  
      → 실패 시 `warnings: ["NEPTUNE_UNAVAILABLE"]` 추가, 진행 계속

11. **특수 플래그 검사** (`special_flags.check(user_id, drug_names)`)  
    → `rds_client.get_user_profile(user_id)` → PostgreSQL `user_profile`  
    → `pregnancy=true`: `query_pregnancy_flags()` → `ddi_pregnancy_flags`  
    → `age >= 65`: `query_age_flags("elderly")` → `ddi_age_flags`

12. **결과 빌드** (`response_builder.build()` × N회)  
    → `view_type="patient"`: `patient_msg` 사용, clinical 전용 필드 null  
    → `view_type="clinical"`: `clinical_msg`, `mechanism`, `pk_pd_summary`, `alternative_drugs`, `reference_url`, `data_updated_at` 포함  
    → severity_level 내림차순 정렬

13. **DDICheckResponse 반환** → Lambda → API Gateway → 프론트엔드

14. **프론트엔드 어댑터 변환** (`ddiAdapter.adaptDDIResponse(raw)`)  
    → `DDICheckResponse` → `DDIViewModel` (아래 3절 참조)

15. **뷰 렌더링**  
    → `patient`: `TrafficLightResult` (severity별 색상 배너 + 카드)  
    → `clinical`: `ClinicalResultSection` (필터 버튼 + `ClinicalResultCard` 테이블 형식)

---

### 예외 경로

| 상황 | 처리 위치 | HTTP 코드 | 응답 |
|------|---------|---------|------|
| API Key 누락/불일치 | `handler._verify_api_key()` | 401 | `{"error_code":"UNAUTHORIZED"}` |
| 잘못된 JSON Body | `handler.lambda_handler()` 파싱 | 400 | `{"error_code":"INVALID_INPUT","message":"..."}` |
| 약물명 미인식 | `drug_mapper.map_drug()` | 404 | `{"error_code":"DRUG_NOT_FOUND"}` |
| 사용자 복용약 없음 | `cache.get_user_medications()` | 200 | `total_pairs=0, results=[]` |
| RDS 연결 실패 | `rds_client.query_ddi_pairs()` | 503 | `{"error_code":"RDS_UNAVAILABLE"}` |
| Neptune 실패 | `_run_parallel_io()` 내부 catch | 200 (경고 포함) | `warnings:["NEPTUNE_UNAVAILABLE"]` 포함 정상 응답 |
| 미처리 예외 | `handler.lambda_handler()` | 500 | `{"error_code":"INTERNAL_ERROR"}` |

---

## 3. 데이터 변환 흐름

| 단계 | 입력 | 출력 | 처리 위치 |
|------|------|------|---------|
| 약물명 정규화 | `"타이레놀"` (한글 상품명) | `"acetaminophen"` (영문 성분명) | `core/drug_mapper.py` `map_drug()` |
| 약물명 정규화 | `"acetaminophen"` (영문) | `"acetaminophen"` (DB ingredient_name_en) | `core/drug_mapper.py` step 4 |
| DDI 조회 | `(drug_name_a, drug_name_b)` 쌍 | `ddi_pairs` 행 (severity_code, clinical_msg, patient_msg 등) | `core/rds_client.py` `query_ddi_pairs()` |
| 캐시 키 생성 | `("acetaminophen", "warfarin")` | `"acetaminophen#warfarin"` (알파벳 정렬) | `core/cache.py` `make_pair_key()` |
| 응답 빌드 | `ddi_pairs` 행 + `view_type` | `DDIResult` 객체 | `core/response_builder.py` `build()` |
| severity 변환 | `L4` → `severity_level=4` | `severity_level` int | `response_builder._SEVERITY_LEVEL` |
| PK/PD 요약 | Neptune `enzyme/role/strength` | `"CYP3A4 억제로 AUC 50~80% 증가"` | `response_builder._build_pk_pd_summary()` |
| reference URL | `reference_no` (숫자) + `source="KR_DUR"` | `https://nedrug.mfds.go.kr/...?itemSeq={no}` | `response_builder._build_reference_url()` |
| 프론트 어댑터 | `max_severity: "L4"` | `overallSeverity: "major"` | `ddiAdapter.ts` `toSeverity()` |
| 프론트 어댑터 | `results[].drug_pair_a/b` | `interactions[].pairLabel: "A + B"` | `ddiAdapter.ts` `adaptDDIResponse()` |
| 프론트 어댑터 | `severity_code: "L3"` | `severityScore: 60` | `ddiAdapter.ts` `SEVERITY_SCORE` |
| 프론트 어댑터 | `source: "KR_DUR"` | `evidenceGrade: "A"` | `ddiAdapter.ts` `EVIDENCE_GRADE_MAP` |
| 메시지 분리 | `"설명 문장. 권고 문장."` | `message: "설명 문장.", action: "권고 문장."` | `ddiAdapter.ts` `splitMessageAction()` |

---

## 4. DB 테이블 및 쿼리 패턴

### 사용 테이블

| 테이블 | 용도 | 조회 파일 |
|--------|------|---------|
| `drug_mapping` | 약물 마스터 (141건) — 한글/영문 성분명, 상품명, alias | `core/drug_mapper.py` |
| `ddi_pairs` | 약물-약물 DDI (315+ pairs) — severity L1-L4, 메시지 | `core/rds_client.py` `query_ddi_pairs()` |
| `drug_supplement_interactions` | 약물-건강기능식품 상호작용 | `core/rds_client.py` `query_supplement_interactions()` |
| `user_profile` | 사용자 프로필 — age, sex, pregnancy, conditions | `core/rds_client.py` `get_user_profile()` |
| `ddi_pregnancy_flags` | 임부금기 약물 (139건) | `core/rds_client.py` `query_pregnancy_flags()` |
| `ddi_age_flags` | 노인주의 약물 | `core/rds_client.py` `query_age_flags()` |
| `ddi_logs` | 조회 로그 (월별 파티션) | `core/rds_client.py` `log_ddi_check()` — 별도 스레드 fire-and-forget |

### 주요 쿼리 패턴

**DDI pair 배치 조회** (`rds_client.query_ddi_pairs`):
```sql
-- (A,B) + (B,A) 양방향 쌍 동적 OR 조건
SELECT drug_name_a, drug_name_b, severity_level, severity_code,
       clinical_msg, patient_msg, mechanism, reference_no, source
FROM   ddi_pairs
WHERE  (drug_name_a = %s AND drug_name_b = %s)
   OR  (drug_name_a = %s AND drug_name_b = %s)
   ...
ORDER BY severity_level DESC
```
> `(col_a, col_b) = ANY(%s::text[][])` 방식은 PostgreSQL 타입 오류로 사용 불가 — 동적 OR 사용

**약물명 해소** (`drug_mapper.map_drug`):
```sql
-- step 1: drug_id
SELECT * FROM drug_mapping WHERE drug_id = %s
-- step 2: 한글 성분명 (한글 입력 시에만)
SELECT * FROM drug_mapping WHERE ingredient_name_kr ILIKE %s LIMIT 1
-- step 3: 한글 상품명 (한글 입력 시에만)
SELECT * FROM drug_mapping WHERE product_name_kr ILIKE %s LIMIT 1
-- step 4: alias 배열 + 영문명 (exact match 우선)
SELECT * FROM drug_mapping
WHERE alias @> ARRAY[%s] OR ingredient_name_en ILIKE %s
ORDER BY CASE WHEN alias @> ARRAY[%s] THEN 0
              WHEN lower(ingredient_name_en) = lower(%s) THEN 1
              ELSE 2 END
LIMIT 1
```

**RDS 연결**: `db_pool.py` — Secrets Manager `ddi/rds/credentials` 1회 로드 → psycopg2 ThreadedConnectionPool (1~5개) → 재시도 3회 + 지수 백오프

---

## 5. API 명세

### POST /ddi-check

**Request**
```json
{
  "user_id": "U002",
  "new_drug_name": "타이레놀",
  "view_type": "patient",
  "include_indirect": true,
  "include_major_only": false,
  "icd10_context": ["I63"]
}
```

**Response (patient view — U002 + 타이레놀 기준)**
```json
{
  "user_id": "U002",
  "new_drug_name": "acetaminophen",
  "total_pairs": 1,
  "max_severity": "L3",
  "results": [
    {
      "drug_pair_a": "acetaminophen",
      "drug_pair_b": "warfarin",
      "severity_code": "L3",
      "severity_level": 3,
      "message": "타이레놀을 하루 2g(500mg 4회) 이상 복용하면 와파린의 혈액 응고 억제 효과가 강해져 출혈 위험이 높아질 수 있습니다. 반드시 의사나 약사에게 알리세요.",
      "mechanism": null,
      "pk_pd_summary": null,
      "alternative_drugs": null,
      "reference_url": null,
      "data_updated_at": null,
      "reference_no": null,
      "source": "manual",
      "cache_hit": true,
      "interaction_type": "direct"
    }
  ],
  "special_flags": {
    "pregnancy_risk": false,
    "pregnancy_detail": null,
    "elderly_risk": false,
    "age_flag": null
  },
  "response_ms": 67,
  "warnings": []
}
```

**오류 응답**

| 코드 | 조건 |
|------|------|
| 400 | Body 누락 또는 필수 필드 없음 (`user_id`, `new_drug_name`, `view_type`) |
| 401 | x-api-key 헤더 누락 또는 불일치 |
| 404 | `DRUG_NOT_FOUND` — 약물명 6단계 해소 모두 실패 |
| 404 | `USER_NOT_FOUND` — DynamoDB 사용자 없음 |
| 503 | `RDS_UNAVAILABLE` — PostgreSQL 연결 실패 |
| 500 | `INTERNAL_ERROR` — 미처리 예외 |

---

## 6. 미구현 / 갭

| 항목 | 설계 의도 | 현재 상태 | 우선순위 |
|------|---------|---------|---------|
| `mechanism` 필드 | clinical view 필수 표시 | DB에 데이터 없어 항상 `null` 반환 | 높음 |
| Bedrock 번역 레이어 | `patient_msg` 없을 때 Claude Haiku로 자동 생성 | 미구현 — `translation_layer/` 디렉터리 없음 | 중간 |
| 익명 모드 | `user_id=null` + `anonymous_drugs[]` 직접 입력 | `user_id` 필수, 익명 미지원 | 중간 |
| Redis 캐시 | Phase 2 — `ddi:pair:{key}` TTL 86400s | `CACHE_BACKEND=dynamodb` 기본값, Redis 미프로비저닝 | 낮음 |
| Neptune 간접 DDI | CYP 효소 경로 기반 간접 상호작용 | 코드 존재하나 연결 불안정 — `NEPTUNE_UNAVAILABLE` warning 처리 중 | 낮음 |
| `icd10_context` 필터 | 요청 시 ICD-10 기반 DDI 우선순위 조정 | Request 모델에 필드 있으나 `pair_engine.run()`에서 미사용 | 낮음 |
| `alternative_drugs` DB | L3+ 시 대체 약물 DB 기반 제공 | `response_builder._DEFAULT_ALTERNATIVES` 하드코딩 | 낮음 |
| CDK 인프라 코드 | `infra/` TypeScript CDK 스택 | 미구현 — 수동 배포 상태 | 중간 |
| `docker-compose.yml` | 로컬 개발 환경 (PostgreSQL + DynamoDB Local) | 미존재 | 낮음 |
| `ddi_logs` 적재 | 조회 로그 PostgreSQL 적재 | `rds_client.log_ddi_check()` 구현됨, `pair_engine`에서 호출 없음 | 낮음 |

---

## 7. 로컬 실행 방법

### 프론트엔드
```bash
cd ddi-ui
cp .env.example .env   # VITE_API_ENDPOINT, VITE_API_KEY 설정
npm install
npm run dev            # http://localhost:5173
```

### 백엔드 (Lambda 로컬 테스트)
```bash
cd ddi-check
pip install -r requirements.txt

# 환경변수 설정
export RDS_SECRET_ARN=<arn>
export DYNAMODB_TABLE_MED=say2-3team-ddi-drugs
export DYNAMODB_TABLE_CACHE=say2-3team-ddi-pairs
export API_KEY_SECRET_ID=ddi/api/key
export AUTH_ENABLED=false   # 로컬 테스트 시 인증 비활성화

python -c "
from handler import lambda_handler
import json
event = {
  'headers': {},
  'body': json.dumps({
    'user_id': 'U002',
    'new_drug_name': '타이레놀',
    'view_type': 'patient'
  })
}
print(lambda_handler(event, None))
"
```

### API 직접 호출 테스트
```bash
cd scripts
python test_api.py          # 7개 시나리오 테스트
python test_drug_mapping.py # 29개 약물명 해소 + 8개 DDI 시나리오
```
