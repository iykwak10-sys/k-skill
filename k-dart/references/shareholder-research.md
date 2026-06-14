# Shareholder Structure Research (지분구조/주요주주 조회)

## Workflow

Use this when the user asks "지분구조", "주요주주", "대주주 지분율", "shareholder structure".

### Step 1 — 대량보유상황보고서 (Fastest Path)

Filter `list.json` with `pblntf_detail_ty=D001` to find recent large-shareholding filings:

```bash
curl -fsS --get 'https://opendart.fss.or.kr/api/list.json' \
  --data-urlencode "crtfc_key=$API_K_DART" \
  --data-urlencode 'corp_code={corp_code}' \
  --data-urlencode 'bgn_de={최근1개월}' \
  --data-urlencode 'end_de={오늘}' \
  --data-urlencode 'pblntf_detail_ty=D001' \
  --data-urlencode 'page_count=5'
```

Check **제출인(flr_nm)** to identify who filed:
- 최대주주 본인 → 에코프로, 삼성전자 등
- 기관/외국인 → 대량보유 변동 발생

**Limitation**: DART API returns metadata only (title, date, filer). Document body needs browser for JS-rendered content.

### Step 2 — Financial Data from 사업보고서

| Data | Endpoint | Query Params |
|------|----------|-------------|
| 재무제표 | `fnlttSinglAcntAll.json` | `bsns_year=2025&reprt_code=11011&fs_div=CFS` |
| 자기주식 | `tesstkAcqsDspsSttus.json` | `bsns_year=2025&reprt_code=11011` |
| 배당 | `alotMatter.json` | `bsns_year=2025&reprt_code=11011` |
| 기업개황 | `company.json` | `corp_code={code}` |

### Step 3 — ddgs 보조 검색

For data not in DART API (foreign ownership %, fund holdings):

Prefer `ddgs.news()` over `ddgs.text()` for timeliness. Write Python script to `/tmp/` and run via terminal.

## Field Reference

### alotMatter.json (배당) — `se` field values

| se 값 | 의미 | 예시 |
|-------|------|------|
| 주당액면가액(원) | 액면가 | 500 |
| (연결)당기순이익(백만원) | 연결 순이익 | 39,371 |
| (별도)당기순이익(백만원) | 별도 순이익 | 92,168 |
| (연결)주당순이익(원) | 연결 EPS | 403 |
| 현금배당금총액(백만원) | 총 현금배당 | 9,782 |
| (연결)현금배당성향(%) | 배당성향 | 24.85 |
| 현금배당수익률(%) | 배당수익률 | 0.05 |
| 주당 현금배당금(원) | DPS | 100 |

### tesstkAcqsDspsSttus.json (자기주식)

| Field | 뜻 | 예시 |
|-------|-----|------|
| `stock_knd` | 주식종류 | 보통주 |
| `bsis_qy` | 기초수량 | 71,865 |
| `change_qy_acqs` | 취득수량 | - |
| `change_qy_dsps` | 처분수량 | 66,022 |
| `trmend_qy` | 기말수량 | 5,843 |
| `stlm_dt` | 결산일 | 2025-12-31 |
| `acqs_mth1/2/3` | 취득방법 | 배당가능이익범위/직접취득/장내직접취득 |

## Complete Research Example (에코프로비엠)

```bash
# 1. corp_code lookup
grep -B5 '에코프로비엠' /tmp/dart_corp/CORPCODE.xml | grep corp_code
# → 01160363

# 2. Company profile
curl -fsS --get 'https://opendart.fss.or.kr/api/company.json' \
  --data-urlencode "crtfc_key=$API_K_DART" \
  --data-urlencode 'corp_code=01160363'

# 3. Recent large shareholding reports
curl -fsS --get 'https://opendart.fss.or.kr/api/list.json' \
  --data-urlencode "crtfc_key=$API_K_DART" \
  --data-urlencode 'corp_code=01160363' \
  --data-urlencode 'bgn_de=20260601' \
  --data-urlencode 'end_de=20260614' \
  --data-urlencode 'pblntf_detail_ty=D001'

# 4. Financial statements (2025 business report)
curl -fsS --get 'https://opendart.fss.or.kr/api/fnlttSinglAcntAll.json' \
  --data-urlencode "crtfc_key=$API_K_DART" \
  --data-urlencode 'corp_code=01160363' \
  --data-urlencode 'bsns_year=2025' \
  --data-urlencode 'reprt_code=11011' \
  --data-urlencode 'fs_div=CFS'

# 5. Treasury shares
curl -fsS --get 'https://opendart.fss.or.kr/api/tesstkAcqsDspsSttus.json' \
  --data-urlencode "crtfc_key=$API_K_DART" \
  --data-urlencode 'corp_code=01160363' \
  --data-urlencode 'bsns_year=2025' \
  --data-urlencode 'reprt_code=11011'
```
