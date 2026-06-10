---
name: ktx-booking
description: Search, reserve, inspect, and cancel KTX or Korail tickets in Korea with the korail2 + pycryptodome Python packages. Use when the user asks for KTX seats, Korail bookings, train changes, or reservation status.
license: MIT
metadata:
  category: travel
  locale: ko-KR
  phase: v1
---

# KTX Booking

## What this skill does

`korail2` 위에 `scripts/ktx_booking.py` helper 를 얹어 KTX/Korail 조회, 예약, 예약 확인, 취소를 처리한다.

최근 Korail 앱의 Dynapath anti-bot 체크 때문에 원본 `korail2` 0.4.0 예제만으로는 `MACRO ERROR` 가 날 수 있다. 이 스킬은 helper 가 `x-dynapath-m-token`, `Sid`, 최신 app version(`250601002`)을 붙여 실제 예매 흐름을 복구하는 것을 전제로 한다.

## When to use

- "서울에서 부산 가는 KTX 찾아줘"
- "코레일 예약 확인해줘"
- "KTX 취소해줘"
- "오전 9시 이후 KTX 중 제일 빠른 거 잡아줘"
- "N카드로 할인 열차 찾아줘"
- "내 N카드 목록 보여줘"
- "N카드 할인 적용해서 예약해줘"

## When not to use

- SRT 예매인 경우
- 실결제 확정까지 자동화해야 하는 경우
- credential 을 평문으로 넣으려는 경우

## Prerequisites

- Python 3.10+
- `python3 -m pip install korail2-ncard pycryptodome`
  - 반드시 **`korail2-ncard`** (fork)를 설치한다. 원본 `korail2`는 Dynapath MACRO ERROR 발생.
  - 주의: `korail2-ncard` 설치는 `korail2` 패키지를 덮어쓰므로, `import korail2`로 그대로 사용한다.

## Required environment variables

- `KSKILL_KTX_ID`
- `KSKILL_KTX_PASSWORD`

### Credential resolution order

1. **이미 환경변수에 있으면** 그대로 사용한다.
2. **에이전트가 자체 secret vault(1Password CLI, Bitwarden CLI, macOS Keychain 등)를 사용 중이면** 거기서 꺼내 환경변수로 주입해도 된다.
3. **`~/.config/k-skill/secrets.env`** (기본 fallback) — plain dotenv 파일, 퍼미션 `0600`.
4. **아무것도 없으면** 유저에게 물어서 2 또는 3에 저장한다.

기본 경로에 저장하는 것은 fallback일 뿐, 강제가 아니다.

## Inputs

- 출발역
- 도착역
- 날짜: `YYYYMMDD`
- 희망 시작 시각: `HHMMSS`
- 인원 수와 승객 유형
- 좌석 선호
- 조회 결과에서 복사한 `train_id`

## Workflow

### 0. Install the package globally when missing

`python3 -c 'import korail2, Crypto'` 가 실패하면 다른 구현으로 우회하지 말고 전역 Python 패키지 설치를 먼저 시도한다.

```bash
python3 -m pip install korail2-ncard pycryptodome
```

### 1. Ensure credentials are available

`KSKILL_KTX_ID`, `KSKILL_KTX_PASSWORD` 환경변수가 설정되어 있는지 확인한다. 없으면 위 credential resolution order에 따라 확보한다.

시크릿이 없다는 이유로 웹사이트를 직접 긁거나 다른 비공식 경로를 찾지 않는다.

### 2. Search first via the helper

항상 helper 를 통해 조회한다.

```bash
python3 scripts/ktx_booking.py search 서울 부산 20260328 090000 --limit 5
```

기본 `--train-type` 은 `ktx` 다. ITX-청춘(예: 남춘천↔용산)·ITX-새마을·무궁화호처럼 KTX 외 노선을 잡으려면 `--train-type` 으로 지정한다.

```bash
python3 scripts/ktx_booking.py search 남춘천 용산 20260503 150000 --train-type itx-cheongchun
```

선택지: `ktx`, `itx-saemaeul`, `mugunghwa`, `nuriro`, `tonggeun`, `itx-cheongchun`, `airport`, `all`.

예약 단계(`reserve`)에서도 같은 `--train-type` 값을 그대로 넘겨야 stable `train_id` 매칭이 깨지지 않는다.

좌석이 없는 열차도 후보에 포함하려면 `--include-no-seats`, 예약 대기 가능한 열차도 같이 보고 싶으면 `--include-waiting-list` 를 붙인다.

### 3. Present the shortlist

예매 전에 항상 아래를 확인한다.

- `index`
- `train_id`
- 출발/도착 시각
- 열차 종류 (`train_type`)
- 일반실/특실 가능 여부
- 예약 대기 가능 여부

### 4. Reserve only after the target train is unambiguous

조회 결과의 `train_id` 를 고른 뒤에만 예약한다. 이 값은 helper 가 열차 번호/운행일/시각/역 코드를 묶어 만든 stable selector 이므로, 재조회 시 같은 열차가 아직 있으면 그대로 잡고 없으면 실패한다.

```bash
python3 scripts/ktx_booking.py reserve 서울 부산 20260328 090000 --train-id <train_id> --seat-option general-first
```

ITX 등 KTX 외 노선을 search 단계에서 골랐다면 reserve 에도 똑같이 `--train-type` 을 넘긴다.

```bash
python3 scripts/ktx_booking.py reserve 남춘천 용산 20260503 150000 --train-id <train_id> --train-type itx-cheongchun --seat-option general-first
```

응답에는 예약번호, 운임, 구입기한이 포함된다. **결제는 자동화하지 않는다.**
좌석이 없을 때는 조회 단계에서 `--include-waiting-list` 를 켜고 예약 단계에서 `--try-waiting` 으로 예약 대기까지 시도할 수 있다.

### 4-1. N-card discounted reservation

N카드 할인을 적용하려면 먼저 보유 N카드 목록을 조회해 카드 번호를 확인한다.

```bash
python3 scripts/ktx_booking.py ncard-list
```

N카드로 할인 열차를 조회한다 (`--ncard-index` 는 `ncard-list` 결과의 순번). `ncard-list` 는 로그/셸 노출을 줄이기 위해 카드 번호를 마스킹해 출력한다.

```bash
python3 scripts/ktx_booking.py ncard-search 대전 서울 20260512 100000 --ncard-index 1 --train-type ktx
```

응답의 `train_id` 를 복사해 `reserve` 에 같은 `--ncard-index` 를 붙여 예약한다.

```bash
python3 scripts/ktx_booking.py reserve 대전 서울 20260512 100000 \
  --train-id <train_id> \
  --ncard-index 1
```

`--ncard-index` 를 지정하면 `--adults` 등 승객 옵션은 무시되고 N카드 승객 1명으로 처리된다. `--ncard-no` 직접 입력도 지원하지만 셸 히스토리에 남을 수 있어 권장하지 않는다. **결제는 자동화하지 않는다.**

N카드 기능은 `korail2-ncard` 패키지가 필요하다. 없으면 해당 커맨드 실행 시 설치 안내가 출력된다.

### 5. Inspect or cancel

취소는 대상 예약을 다시 조회해 식별한 뒤에만 진행한다.

```bash
python3 scripts/ktx_booking.py reservations
```

```bash
python3 scripts/ktx_booking.py cancel <reservation_id>
```

## Done when

- 조회면 열차 후보가 정리되어 있다
- 예약이면 예약 결과와 제한 시간이 확인되어 있다
- 취소면 어떤 예약을 취소했는지 남아 있다

## ⚠️ Known pitfalls

### 1. `scripts/ktx_booking.py` helper script may not exist locally

k-skill 번들이 미설치 상태면 `scripts/ktx_booking.py` helper가 존재하지 않는다. 실행 전에 반드시 확인한다:

```bash
python3 scripts/ktx_booking.py --help 2>/dev/null || echo "HELPER_MISSING"
```

helper가 없으면 **직접 `korail2` API로 대체**한다. 아래 참조.

### 2. Dynapath MACRO ERROR

`korail2` 원본 패키지(0.4.0)는 Korail 안티봇 체크(Dynapath)에서 `MACRO ERROR` 발생. **반드시 `korail2-ncard` fork를 설치**해야 한다:

```bash
python3 -m pip install korail2-ncard pycryptodome
```

설치 후에도 import는 `from korail2 import Korail, AdultPassenger`로 동일하게 사용한다.

### 3. Train 객체 속성명 차이

`korail2-ncard` fork의 Train 객체는 원본 `korail2`와 속성명이 다르다:

| 속성 | 타입 | 의미 |
|------|------|------|
| `general_seat` | `str` (→int) | 일반실 잔여석. `int(v) > 0`이면 예약 가능 |
| `special_seat` | `str` (→int) | 특실 잔여석 |
| `wait_reserve_flag` | `str` | `"1"`=예약대기 가능, `"-2"`=불가 |
| `reserve_possible` | `str` | `"Y"`=예약 가능 |
| `train_no` | `str` | 열차 번호 (ex: `"075"`) |
| `train_type_name` | `str` | `"KTX"`, `"KTX-산천"` 등 |
| `dep_time` / `arr_time` | `str` | `"065700"` 형식 |**

주의: `general_seat_available`, `special_seat_available`, `waiting_available` 같은 bool 속성은 존재하지 않는다. `general_seat`의 int값으로 판단한다.

### 4. `Station` 클래스 없음

`from korail2 import Station` 은 존재하지 않는다. 역명은 **일반 문자열**로 전달한다.

```python
# ⚠️ 안 됨
from korail2 import Station
dep = Station.search('오송')

# ✅ 됨
trains = k.search_train_allday('오송', '부산', date='20260614', ...)
```

### 5. `search_train_allday` 사용

`search_train`보다 **`search_train_allday`**가 하루 전체 열차를 반환하므로 권장.

```python
trains = k.search_train_allday('오송', '부산', date='20260614', time='000000',
                                passengers=[AdultPassenger(1)])
```

### 6. credentials 환경변수 주입

`~/.config/k-skill/secrets.env` 파일은 Python에서 직접 읽어 `os.environ`에 주입해야 한다. 셸에서 `export` 없이 python3를 실행하면 환경변수가 비어 있다.

## Failure modes

- 로그인 실패
- 매진
- Korail anti-bot 규칙 변경 (Dynapath 업데이트 시)`

## Notes

- `scripts/ktx_booking.py` 는 upstream `korail2` anti-bot 회귀를 보완하는 helper 다
- `korail2` 는 KTX/Korail 전용 표면이라 train type 과 passenger model 이 분명하다
- 결제 완료까지는 자동화하지 않는다
- aggressive polling 은 피한다
