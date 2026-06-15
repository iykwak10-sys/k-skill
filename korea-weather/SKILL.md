---
name: korea-weather
description: 한국 날씨를 기상청 단기예보 조회서비스와 프록시 경유로 조회해 요약한다.
license: MIT
metadata:
  category: weather
  locale: ko-KR
  phase: v1
---

# Korea Weather

## What this skill does

기상청 단기예보 조회서비스를 `k-skill-proxy` 경유로 조회해서 한국 날씨를 요약한다.
사용자는 개인 OpenAPI key를 직접 발급할 필요가 없고, proxy 서버에만 `KMA_OPEN_API_KEY` 를 둔다.

## When to use

- "서울 시청 근처 지금 날씨 어때?"
- "부산 날씨 알려줘"
- "위도/경도 기준으로 한국 단기예보 보고 싶어"

## Prerequisites

- optional: `jq`
- optional: `KSKILL_PROXY_BASE_URL` (self-host·별도 프록시를 쓸 때만 설정. 비우면 기본 hosted `https://k-skill-proxy.nomadamas.org` 를 사용한다.)

## Required environment variables

- 없음. `KSKILL_PROXY_BASE_URL` 은 선택 사항이며, 비우면 기본 hosted `https://k-skill-proxy.nomadamas.org` 를 사용한다.

사용자가 공공데이터포털 기상청 API key를 직접 다룰 필요는 없다. `/v1/korea-weather/forecast` route는 기본 hosted proxy에서 호출하고, upstream `KMA_OPEN_API_KEY` 는 proxy 서버에서만 관리한다. 별도 proxy를 쓰는 경우에만 `KSKILL_PROXY_BASE_URL` 을 설정한다.

## Inputs

- 격자 좌표: `nx`, `ny`
- 또는 위도/경도: `lat`, `lon`
- 선택 사항: `baseDate`, `baseTime`

`baseDate` / `baseTime` 을 생략하면 proxy 가 KST 기준 최신 단기예보 발표 시각을 자동으로 고른다.

## Workflow

### 1. Resolve the proxy base URL

`KSKILL_PROXY_BASE_URL` 이 있으면 그 값을 사용하고, 없거나 비어 있으면 기본 hosted proxy `https://k-skill-proxy.nomadamas.org` 를 사용한다.

### 2. Query the short-term forecast endpoint

격자 좌표가 이미 있으면 그대로 넣고, 위도/경도만 있으면 proxy 에 그대로 넘긴다.

```bash
BASE="${KSKILL_PROXY_BASE_URL:-https://k-skill-proxy.nomadamas.org}"
curl -fsS --get "${BASE}/v1/korea-weather/forecast" \
  --data-urlencode 'lat=37.5665' \
  --data-urlencode 'lon=126.9780'
```

격자 좌표 예시:

```bash
BASE="${KSKILL_PROXY_BASE_URL:-https://k-skill-proxy.nomadamas.org}"
curl -fsS --get "${BASE}/v1/korea-weather/forecast" \
  --data-urlencode 'nx=60' \
  --data-urlencode 'ny=127' \
  --data-urlencode 'baseDate=20260405' \
  --data-urlencode 'baseTime=0500'
```

### 3. Summarize the response conservatively

가능하면 아래 항목만 먼저 요약한다.

- `TMP`: 기온
- `SKY`: 하늘상태
- `PTY`: 강수형태
- `POP`: 강수확률
- `PCP`: 강수량
- `SNO`: 적설
- `REH`: 습도
- `WSD`: 풍속

응답에는 조회 시점과 `baseDate` / `baseTime` 도 함께 적는다.

## Done when

- 요청 위치의 단기예보 응답이 정리되어 있다
- 조회 시점과 예보 발표 시각이 명시되어 있다
- upstream key가 클라이언트에 노출되지 않았다

## Failure modes

- proxy upstream key 미설정 또는 hosted/self-host route 장애
- `nx` / `ny` 또는 `lat` / `lon` 이 불완전한 경우
- 기상청 quota 초과 또는 upstream 장애
- 선택한 발표 시각에 아직 예보가 준비되지 않은 경우
- **Proxy 429 (Rate Limited)**: k-skill-proxy가 429 Too Many Requests를 반환하면 일시적 과부하 상태. 아래 **Fallback Workflow**로 즉시 전환한다. 재시도(지수 백오프) 후에도 실패하면 Naver Weather로 직접 조회한다.

## Fallback Workflow (Proxy 실패 시)

proxy가 429 또는 기타 오류로 실패하면, browser로 Naver Weather Search에 접속한다.

### 1. Naver Weather Search 열기

```bash
https://search.naver.com/search.naver?query={도시명}+날씨
```

예시:
- `세종시 날씨` → `https://search.naver.com/search.naver?query=세종시+날씨`
- `부산 날씨` → `https://search.naver.com/search.naver?query=부산+날씨`

### 2. Browser로 데이터 수집

browser_navigate → browser_snapshot 순서로 접근한다.

Naver 페이지에서 확인 가능한 정보:
- **현재 날씨**: 기온, 체감온도, 습도, 풍향/풍속
- **시간별 예보**: 1시간 간격 날씨/기온 (오늘 ~ 글피)
- **일별 요약 (주간전망 탭)**: 
  - `tab "전망"` 클릭 → `tab "주간전망"` 에서 10일치 오전/오후 날씨 + 강수확률 + 최저/최고기온
  - 네이버 주간예보는 `오늘: 6.04. 오전 10% 맑음 오후 60% 구름많고 가끔 소나기 최저기온 19° 최고기온 29°` 형식
- **미세먼지/초미세먼지**: 좋음/보통/나쁨
- **자외선 지수**: 낮음/보통/높음/매우높음

탭 전환 시 browser_click(ref) → browser_snapshot 순서로 데이터를 읽는다.

| tab | 클릭할 ref |
|-----|-----------|
| 오늘 | `오늘` 링크 ref (selected 시 오늘의 현재+시간별) |
| 내일 | `내일` 링크 ref (오전/오후 요약 + 시간별) |
| 모레 | `모레` 링크 ref (오전/오후 요약 + 시간별) |
| 전망 | `전망` 링크 ref → 다시 `주간전망` 탭 ref (10일 outlook) |

### 3. 요약 시 출력할 항목 (우선순위)

- 날짜/요일
- 오전 날씨 + 강수확률
- 오후 날씨 + 강수확률
- 최저기온 / 최고기온
- 미세먼지 (있으면)
- 특이사항 (소나기, 돌풍, 일교차 등)

## Notes

- 공식 API는 `nx` / `ny` 격자를 쓰지만, proxy 는 `lat` / `lon` 도 받아 내부에서 격자로 변환한다.
- 단기예보 category 는 `TMP`, `SKY`, `PTY`, `POP`, `PCP`, `SNO`, `REH`, `WSD` 등을 중심으로 본다.
- proxy 운영/환경변수 설정은 `docs/features/k-skill-proxy.md` 를 참고한다.
- Naver Weather 검색으로도 기상청 원천 데이터를 동일하게 조회 가능하다. 기상청 단기예보는 KST 기준 06:00, 12:00, 18:00, 24:00 발표.
