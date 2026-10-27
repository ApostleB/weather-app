# 날씨 지도 (weather-app) 설계

- 작성일: 2026-10-27
- 상태: 승인됨

## 목표

Windy처럼 지도가 메인인 날씨 웹. 동아시아 바람 입자 애니메이션 위에서 지점을 클릭하면 기상청 예보(현재·시간별·주간)를 보여준다.

- 운영비 0원(무료 데이터만): 기상청 공공데이터포털 API, NOAA GFS, OpenFreeMap
- 개인 서버 `bytebard.cloud`의 `/weather` 경로로 서빙 (Docker·Cloudflare 미사용)
- 웹 먼저, 이후 Capacitor로 앱 확장 (MVP 범위 밖)

## 범위

MVP에 포함:

- 전체 화면 지도 + 동아시아(북위 20~50°, 동경 110~150°) 10m 바람 입자 애니메이션
- 시간 슬라이더: 현재~+72시간, 3시간 간격(25개 시점)
- 지도 클릭 → 상세 패널: 현재(초단기실황), 시간별(단기예보, 최대 5일), 주간(중기예보 4~10일)
- "내 위치" 버튼 (Geolocation)
- 출처 표기: 기상청, NOAA GFS, OpenFreeMap/OpenStreetMap

MVP에서 제외: 모바일 앱, 특보·레이더·미세먼지, 기온·강수 등 바람 외 지도 레이어, 지역 검색, GitHub Actions 자동 배포.

## 구조

```
weather-app/                 pnpm workspace
├─ web/                      React + Vite + TypeScript + MapLibre GL (base: /weather/)
├─ server/                   Hono on Node 24, 127.0.0.1:3090
├─ wind/                     Python 배치 (GFS → 바람 PNG)
├─ deploy/nginx/             weather.conf + install.sh
├─ scripts/                  deploy.sh (로컬), remote-deploy.sh (서버)
└─ ecosystem.config.cjs      pm2 앱: weather (서버), weather-wind (배치)
```

서버(Hono)의 경로 (앱이 `/weather` prefix를 그대로 받는다):

| 경로 | 내용 |
|---|---|
| `/weather/` | web 빌드 결과(정적 파일), SPA fallback |
| `/weather/api/forecast?lat=&lon=` | 기상청 프록시 + 캐시 |
| `/weather/data/wind/*` | 바람 `latest.json`, PNG |
| `/weather/healthz` | 상태 확인 |

## 기상청 데이터

공공데이터포털(data.go.kr) 방식: `https://apis.data.go.kr/1360000/...`, 파라미터 `serviceKey`(Decoding 키를 URL 인코딩해 전송), `dataType=JSON`. 키는 서버 `.env`의 `KMA_SERVICE_KEY`에만 있다.

> 단기예보 가이드(2610)에는 `apihub.kma.go.kr` + `authKey` 예시가 있으나, 공공데이터포털 키를 쓰므로 `apis.data.go.kr/1360000/VilageFcstInfoService_2.0`을 사용한다. 구현 시 실호출로 확인한다.

| 용도 | API | 조회 키 | 발표·제공 시각 (KST) |
|---|---|---|---|
| 현재 | `VilageFcstInfoService_2.0/getUltraSrtNcst` | nx, ny | 매시 정각 발표, HH:10 이후 제공 |
| 시간별 | `VilageFcstInfoService_2.0/getVilageFcst` | nx, ny | 02·05·08·11·14·17·20·23시 발표, +10분 이후 제공 |
| 주간 날씨 | `MidFcstInfoService/getMidLandFcst` | 육상 regId (예 11B00000) | 06·18시 발표 (최근 24시간만 제공) |
| 주간 기온 | `MidFcstInfoService/getMidTa` | 기온 regId (예 11B10101) | 06·18시 발표 |

- 발표시각 계산: 현재 시각에서 "제공 시각이 지난 가장 최근 발표"를 고른다(자정 이전 날짜로 넘어가는 경우 포함). 응답이 `03 NODATA_ERROR`면 직전 발표로 한 번 재시도한다.
- 단기예보는 한 번에 받는다(`numOfRows=1000`).
- 중기 발표시각 tmFc: 06:10 이전이면 전날 18시, 18:10 이전이면 당일 06시, 그 외 당일 18시(다른 API와 같이 10분 여유). 06시 발표는 4일 후부터, 18시 발표는 5일 후부터 값이 있다.
- 단기예보 연장기간 판정은 날짜가 아니라 값 형식으로 한다: 그 시각의 PCP가 숫자 한 자리(`0`~`3`)면 정성 코드 구간이며, 이때 PCP·SNO·WSD를 코드표 문구로 바꾼다(실측: 14시 발표에서 +3일은 정량값, +4일부터 코드).
- `daily`의 단기예보 날짜는 발표일과 TMN/TMX가 있는 날만 쓰고, 그 이후는 중기예보로 채운다. 발표일에 TMN/TMX가 없으면 남은 시간 TMP의 최저/최고로 대신한다.

### 위경도 ↔ 격자

가이드의 Lambert Conformal Conic C 예제를 TypeScript로 옮긴다(Re=6371.00877km, grid=5km, slat1=30, slat2=60, olon=126, olat=38, xo=43, yo=136). 격자 범위 1≤nx≤149, 1≤ny≤253. 테스트 기준값: `(126.929810, 37.488201) → (59, 125)`.

### 중기 지역코드 매핑

빌드 전 스크립트가 가이드 엑셀 두 개를 `server/src/data/*.json`으로 변환한다(엑셀 원본은 `data-src/`에 보관).

1. 클릭 위치에서 가장 가까운 읍면동 행(격자 엑셀, 약 3,800행: 1·2·3단계 이름, 격자 X/Y, 위경도)을 찾는다.
2. 1단계(시도) → 육상 regId. 강원은 2단계(시군구)로 영서(11D10000)/영동(11D20000)을 나눈다.
3. 2단계(시군구) 이름 → 중기기온 엑셀(약 240행)의 구역명과 매칭해 기온 regId를 얻는다. 매칭이 안 되면 같은 시도의 대표 도시 코드를 쓴다.
4. 가장 가까운 읍면동이 너무 멀면(20km 초과) 남한 밖으로 보고 "국내만 상세 예보 지원"을 응답한다.

### 응답 형식 `/weather/api/forecast`

세 API를 병렬로 호출하고, 일부가 실패해도 나머지는 응답한다.

```ts
{
  location: { lat: number; lon: number; nx: number; ny: number; name: string } // name: "서울특별시 종로구 사직동"
  current: { observedAt: string; temp: number | null; humidity: number | null; rain1h: number | null; windSpeed: number | null; windDir: number | null; pty: number | null } | null
  hourly: { time: string; temp: number | null; sky: number | null; pty: number | null; pop: number | null; humidity: number | null
            windDir: number | null; windSpeed: number | null; windText: string | null; pcp: string | null; sno: string | null }[]
            // 정성 코드 구간이면 windSpeed=null, windText="약한 바람" 등
  daily: { date: string; min: number | null; max: number | null; amWf: string | null; pmWf: string | null; amPop: number | null; pmPop: number | null }[]
  issued: { current?: string; hourly?: string; mid?: string } // 사용한 발표시각
  stale: boolean                      // 하나라도 만료된 캐시로 대신했으면 true
  errors: { current?: string; hourly?: string; daily?: string }
}
```

- `daily`는 단기예보(오늘~+3일: TMN/TMX, 대표 SKY/PTY/POP 오전·오후)와 중기예보(+4~+10일)를 합친다.
- PCP·SNO 같은 범주형 값은 가이드 표대로 문자열로 내려준다("1mm 미만", "30.0~50.0mm"). -900 이하, +900 이상은 결측(null)으로 처리한다.

### 캐시

- 메모리 LRU(최대 5,000개). 키 `(API, nx/ny 또는 regId)`, 값에 사용한 발표시각을 함께 저장한다. 같은 키로 동시에 들어온 요청은 기상청 호출 하나를 공유한다.
- 유효기간은 다음 발표의 제공 시각까지(NODATA로 직전 발표를 썼으면 최소 5분 뒤 재시도). 호출 실패 시 만료된 캐시가 있으면 그것을 쓰고 `stale: true`를 표시한다.
- 같은 격자는 발표당 1회만 호출하므로 하루 호출 한도(개발계정) 안에서 운영할 수 있다. 한도 초과(`22`)는 로그로 남긴다.

## 바람 지도

### 배치 `wind/` (pm2 앱 `weather-wind`)

- pm2 `cron_restart`로 매시 15분에 실행되고, 끝나면 종료한다(`autorestart: false`).
- 가장 최근 GFS 사이클(00/06/12/18Z)부터 확인한다. 처리 완료된 사이클이면 바로 종료하고, 아직 공개 전이면 이전 사이클을 쓴다.
- NOMADS grib filter로 10m UGRD/VGRD, 동경 110~150°, 북위 20~50°만 받는다(`filter_gfs_0p25.pl`, 0.25° → 161×121 격자). 예보시간 f000~f072를 3시간 간격으로 받는다(25개, 요청 사이 간격을 두어 NOMADS 제한을 지킨다).
- Python venv(`eccodes`, `numpy`, `pillow`)로 디코딩해 프레임마다 PNG 1장을 만든다. R=U, G=V를 프레임별 min/max로 0~255 정규화한다.
- `data/wind/<cycle>/`에 쓰고 25장이 다 되면 `latest.json`을 임시 파일 → rename으로 교체한다. 실패하면 이전 데이터가 그대로 남는다. 최근 2개 사이클만 보관한다.

`latest.json`:

```ts
{
  cycle: string             // "2026102706" (UTC)
  bbox: [110, 20, 150, 50]  // [west, south, east, north]
  width: 161; height: 121
  frames: { time: string; file: string; uMin: number; uMax: number; vMin: number; vMax: number }[]
}
```

### 프론트 바람 레이어

- MapLibre GL + OpenFreeMap 배경 스타일(키 없음).
- 지도 위 Canvas 2D 오버레이로 입자 애니메이션을 그린다: 입자는 위경도로 들고 바람 PNG를 쌍선형 보간해 이동하며, 이전 프레임을 서서히 지워 잔상을 만든다. 속도 구간별 색으로 그린다. 지도 이동·줌 중에는 숨기고, 끝나면 입자를 초기화한다. (WebGL GPGPU 방식보다 단순하고, 동아시아 범위·수천 개 입자로는 성능이 충분하다)
- 풍속 배경: 같은 PNG로 풍속 색상 이미지를 만들어 MapLibre image source로 깐다. 위경도 격자를 그대로 펴면 메르카토르에서 위도가 최대 약 2° 어긋나므로, 행을 메르카토르 y 등간격으로 다시 샘플링한다.
- 시간 슬라이더로 프레임을 바꾼다. 처음에는 현재 시각에 가장 가까운 프레임을 고른다.

## 화면

- 전체 화면 지도, 우상단 "내 위치" 버튼, 하단 시간 슬라이더(시각 표시, 이전/다음), 하단 출처 표기.
- 클릭 위치에 마커를 찍고 패널을 연다(데스크톱: 우측 패널, 모바일: 하단 시트).
  - 현재: 기온, 하늘·강수 형태, 습도, 바람(16방위 + m/s)
  - 시간별: 가로 스크롤 목록(시각, 아이콘, 기온, 강수확률, 바람)
  - 주간: 날짜별 오전/오후 날씨, 최저/최고 기온
- 16방위 변환은 가이드 공식(`floor((deg + 22.5*0.5) / 22.5)`)을 따른다.

## 에러 처리

| 상황 | 처리 |
|---|---|
| lat/lon 누락·숫자 아님·지구 좌표 범위 밖 | 400 + 메시지 |
| 남한 밖 (가장 가까운 읍면동 20km 초과, 또는 격자 범위 밖) | 200 + `location: null`, 기상청 호출 없음, 프론트에서 "국내만 상세 예보 지원" |
| 기상청 `resultCode≠00` / 네트워크 오류 / 10초 타임아웃 | 해당 섹션 `errors.*`에 사용자용 문구, 만료 캐시가 있으면 `stale: true`로 대체 |
| `03 NODATA` | 직전 발표시각으로 1회 재시도 |
| 바람 `latest.json` 없음·오류 | 바람 레이어와 슬라이더를 숨기고 "바람 데이터 준비 중" 표시, 나머지 기능은 정상 |
| 바람 데이터 사이클이 14시간 넘게 오래됨 (정상 운영에서도 공개 지연 때문에 최대 약 11.5시간) | 슬라이더에 기준 사이클 시각과 "업데이트 지연" 표시 |
| `KMA_SERVICE_KEY` 없음 | 서버 시작 시 경고 로그, `/api/forecast`는 503 |

## 테스트

- vitest (server):
  - 격자 변환 기준값 (가이드 예시, 엑셀의 서울특별시 행)
  - 발표시각 계산(정시 직전·직후, 자정, 중기 06/18 경계)
  - 단기·중기 응답 파싱(범주값, 결측값)
  - 지역코드 매핑(서울, 강릉=영동, 춘천=영서, 제주, 바다 위 좌표)
  - `/api/forecast` 라우트(fetch 목킹: 부분 실패, stale, 캐시 적중, 동시 요청 합치기)
- vitest (web): 16방위 변환, 프레임 선택 등 순수 함수.
- pytest (wind): 작은 합성 U/V 배열 → PNG 인코딩·min/max 왕복, `latest.json` 원자 교체.

## 배포 (coffee-note 방식)

- 저장소: `github.com/ApostleB/weather-app` (public), 브랜치 `master`.
- 서버 경로 `/home/rocky/weather-app`, `.env`는 서버에 직접 둔다(저장소에 넣지 않음).
- `scripts/deploy.sh` (로컬) → 서버에서 git fetch/checkout → `scripts/remote-deploy.sh`:
  `pnpm install --frozen-lockfile` → `pnpm build` → wind venv 준비(`python3 -m venv`, `pip install -r wind/requirements.txt`) → `pm2 startOrReload ecosystem.config.cjs --update-env` → `pm2 save` → `curl http://127.0.0.1:3090/weather/healthz`.
- nginx: 기존 `/etc/nginx/conf.d/bytebard.cloud.conf`의 443 server 블록에 `include /etc/nginx/bytebard.cloud.d/*.conf;` 한 줄만 추가하고, `deploy/nginx/weather.conf`를 `/etc/nginx/bytebard.cloud.d/weather.conf`로 설치한다.
  - `location = /weather { return 301 /weather/; }`
  - `location ^~ /weather/ { proxy_pass http://127.0.0.1:3090; ...표준 프록시 헤더 }`
  - `deploy/nginx/install.sh`(root로 1회 실행): 백업 → 설치 → `nginx -t` → `systemctl reload nginx`, 실패하면 복구한다. 인증서는 기존 `bytebard-cloud`를 사용한다.

## 환경변수

| 이름 | 위치 | 설명 |
|---|---|---|
| `KMA_SERVICE_KEY` | `.env` | 공공데이터포털 일반 인증키(Decoding) |
| `KMA_APIHUB_AUTH_KEY` | `.env` | 예약(MVP 미사용) |
| `PORT` | ecosystem (3090) | 서버 포트 |
| `WIND_DATA_DIR` | ecosystem (`./data/wind`) | 바람 데이터 경로 |
