# City Weather URLs for Naver Fallback

Naver Weather 검색 URL 패턴:
`https://search.naver.com/search.naver?query={도시}+날씨`

## 주요 도시

| 도시 | 검색어 | URL |
|------|--------|-----|
| 서울 | 서울+날씨 | `https://search.naver.com/search.naver?query=서울+날씨` |
| 부산 | 부산+날씨 | `https://search.naver.com/search.naver?query=부산+날씨` |
| 대구 | 대구+날씨 | `https://search.naver.com/search.naver?query=대구+날씨` |
| 인천 | 인천+날씨 | `https://search.naver.com/search.naver?query=인천+날씨` |
| 광주 | 광주+날씨 | `https://search.naver.com/search.naver?query=광주+날씨` |
| 대전 | 대전+날씨 | `https://search.naver.com/search.naver?query=대전+날씨` |
| 울산 | 울산+날씨 | `https://search.naver.com/search.naver?query=울산+날씨` |
| 세종 | 세종시+날씨 | `https://search.naver.com/search.naver?query=세종시+날씨` |
| 경기 | 경기도+날씨 | `https://search.naver.com/search.naver?query=경기도+날씨` |
| 강원 | 강원도+날씨 | `https://search.naver.com/search.naver?query=강원도+날씨` |
| 충북 | 충청북도+날씨 | `https://search.naver.com/search.naver?query=충청북도+날씨` |
| 충남 | 충청남도+날씨 | `https://search.naver.com/search.naver?query=충청남도+날씨` |
| 전북 | 전라북도+날씨 | `https://search.naver.com/search.naver?query=전라북도+날씨` |
| 전남 | 전라남도+날씨 | `https://search.naver.com/search.naver?query=전라남도+날씨` |
| 경북 | 경상북도+날씨 | `https://search.naver.com/search.naver?query=경상북도+날씨` |
| 경남 | 경상남도+날씨 | `https://search.naver.com/search.naver?query=경상남도+날씨` |
| 제주 | 제주도+날씨 | `https://search.naver.com/search.naver?query=제주도+날씨` |

## KMA 중기예보 지역별 URL (browser 직접 접근 시)

KMA mid-term forecast는 `stnId` 파라미터로 지역 전환 가능:

| 지역 | URL |
|------|-----|
| 전국 | `https://www.weather.go.kr/w/forecast/overall/mid-term.do?stnId1=108` |
| 서울/경기 | `https://www.weather.go.kr/w/forecast/overall/mid-term.do?stnId1=109` |
| 대전/세종/충남 | `https://www.weather.go.kr/w/forecast/overall/mid-term.do?stnId1=133&stnId2=131` |
| 충북 | `https://www.weather.go.kr/w/forecast/overall/mid-term.do?stnId1=131` |
| 강원 | `https://www.weather.go.kr/w/forecast/overall/mid-term.do?stnId1=105` |
| 전북 | `https://www.weather.go.kr/w/forecast/overall/mid-term.do?stnId1=146` |
| 전남 | `https://www.weather.go.kr/w/forecast/overall/mid-term.do?stnId1=156` |
| 경북 | `https://www.weather.go.kr/w/forecast/overall/mid-term.do?stnId1=143` |
| 경남 | `https://www.weather.go.kr/w/forecast/overall/mid-term.do?stnId1=159` |
| 제주 | `https://www.weather.go.kr/w/forecast/overall/mid-term.do?stnId1=184` |
