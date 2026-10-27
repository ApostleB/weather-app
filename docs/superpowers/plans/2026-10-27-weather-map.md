# 날씨 지도 (weather-app) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `https://bytebard.cloud/weather`에서 동아시아 바람 애니메이션 지도를 보여주고, 클릭한 지점의 기상청 예보(현재·시간별·10일)를 패널로 보여준다.

**Architecture:** pnpm workspace. `server/`(Hono, Node)가 web 빌드 결과·바람 데이터 정적 파일·기상청 프록시(`/weather/api/forecast`, 메모리 캐시)를 한 포트(3090)에서 서빙한다. `wind/`(Python, pm2 cron)가 NOAA GFS 10m 바람을 받아 PNG 프레임으로 만든다. `web/`(React + Vite + MapLibre)이 Canvas 2D 입자 애니메이션과 풍속 배경, 예보 패널을 그린다. 서버 nginx가 `/weather/`를 3090으로 넘긴다.

**Tech Stack:** Node 22+(서버는 24), pnpm 11, TypeScript 5.9, Hono 4 + @hono/node-server 2, Vitest 5, React 19, Vite 8, MapLibre GL 6, Python 3.12 + eccodes 2.50 + numpy + pillow + pytest, pm2, nginx.

설계 문서: `docs/superpowers/specs/2026-10-27-weather-map-design.md`

## Global Constraints

- 모든 경로는 `/weather` prefix 아래: 페이지 `/weather/`, API `/weather/api/forecast`, 바람 `/weather/data/wind/`, 상태 `/weather/healthz`.
- 서버는 `127.0.0.1:3090`에만 listen. pm2 앱 이름은 `weather`, `weather-wind`.
- 기상청은 공공데이터포털 방식: `https://apis.data.go.kr/1360000/...`, `serviceKey`, `dataType=JSON`. 키는 `.env`의 `KMA_SERVICE_KEY`(포털의 Encoding 키도 Decoding 키도 받는다). 키를 로그·응답·커밋에 절대 넣지 않는다.
- 시각 기준은 KST(UTC+9). 바람 데이터 시각만 UTC(`...Z`).
- 바람 범위 bbox `[110, 20, 150, 50]`(west, south, east, north), GFS 0.25° → 161×121, 예보 f000~f072 3시간 간격(25프레임).
- 무료 데이터만: 기상청, NOAA GFS(NOMADS), OpenFreeMap(`https://tiles.openfreemap.org/styles/liberty`). Cloudflare·Docker 사용하지 않음.
- 서버 경로 `/home/rocky/weather-app`, 저장소 `https://github.com/ApostleB/weather-app`(public), 브랜치 `master`.
- 사용자에게 보이는 문구는 한국어.
- 커밋 메시지 끝에 빈 줄 후 `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`를 붙인다(아래 커밋 명령은 두 번째 `-m`으로 넣는다).

## 이미 저장소에 있는 것

- `.env`(커밋 안 됨, `KMA_SERVICE_KEY` 채워짐), `.env.example`, `.gitignore`, `README.md`, 설계 문서
- `data-src/kma-grid-latlon-2610.xlsx` (단기예보 격자·위경도, 3,839행), `data-src/kma-mid-temp-regions-2512.xlsx` (중기 구역코드, 240행)
- `server/test/fixtures/{ncst,vilage,midLand,midTa}.json`: 2026-10-27 실제 응답(초단기실황 15시, 단기예보 14시 발표, 중기 06시 발표, 서울 nx=60 ny=127, 육상 11B00000, 기온 11B10101)
- `wind/tests/fixtures/gfs-10m-f003.grb2`: NOMADS에서 받은 동아시아 10m U/V GRIB2 (161×121, jScansPositively=1)

## File Structure

```
package.json, pnpm-workspace.yaml, tsconfig.base.json      워크스페이스
ecosystem.config.cjs                                       pm2 (weather, weather-wind)
server/
  package.json, tsconfig.json, tsconfig.build.json
  scripts/build-regions.ts      엑셀 → src/data/*.json (수동 실행)
  src/index.ts                  진입점: .env 로드, 의존성 조립, listen
  src/app.ts                    Hono 라우트 (healthz, api, 정적)
  src/static.ts                 root 밖 접근을 막는 정적 파일 읽기
  src/types.ts                  API 응답 타입 (web도 import)
  src/cache.ts                  TtlCache: TTL + stale + 동시요청 합치기 + LRU
  src/forecast.ts               예보 서비스: 위치→격자·구역, 3개 API 병렬, 응답 조립
  src/regions.ts                가까운 읍면동, 중기 육상/기온 구역코드
  src/data/towns.json, midTempRegions.json   (생성물, 커밋)
  src/kma/grid.ts               위경도 → 격자 (Lambert)
  src/kma/baseTime.ts           발표시각 계산
  src/kma/client.ts             기상청 HTTP + 오류 봉투 해석
  src/kma/parse.ts              응답 → Current/Hourly/Daily
  test/*.test.ts
wind/
  requirements.txt, requirements-dev.txt, pytest.ini
  wind_job.py                   GFS 다운로드·디코드·PNG·latest.json
  tests/test_wind_job.py
web/
  package.json, tsconfig.json, vite.config.ts, index.html
  src/main.tsx, App.tsx, styles.css
  src/api.ts                    /api/forecast 호출
  src/format.ts                 16방위, 날씨 문구·아이콘, 시각 포맷
  src/map/MapView.tsx           MapLibre 지도 + 클릭 + 마커
  src/wind/field.ts             PNG→바람장, 쌍선형 샘플, 프레임 선택, 풍속 색상 이미지
  src/wind/particles.ts         입자 생성·이동 (순수 함수)
  src/wind/WindParticles.ts     Canvas 2D 애니메이션 (DOM)
  src/wind/useWind.ts           latest.json/프레임 로드, 배경 레이어 갱신
  src/components/TimeSlider.tsx, ForecastPanel.tsx
deploy/nginx/weather.conf, deploy/nginx/install.sh
scripts/deploy.sh, scripts/remote-deploy.sh
```

---

### Task 1: 워크스페이스와 서버 뼈대 (`/weather/healthz`)

**Files:**
- Create: `package.json`, `pnpm-workspace.yaml`, `tsconfig.base.json`
- Create: `server/package.json`, `server/tsconfig.json`, `server/tsconfig.build.json`
- Create: `server/src/types.ts`, `server/src/app.ts`, `server/src/index.ts`
- Modify: `.gitignore`
- Test: `server/test/app.test.ts`

**Interfaces:**
- Produces: `createApp(deps: AppDeps): Hono` (`server/src/app.ts`), `AppDeps = { forecast: ForecastProvider | null; webDir?: string; windDir?: string }`, `ForecastProvider = { get(lat: number, lon: number): Promise<ForecastResponse> }`, 그리고 `server/src/types.ts`의 `ForecastResponse`, `Location`, `Current`, `Hourly`, `Daily` (이후 모든 태스크와 web이 사용).

- [ ] **Step 1: 루트 워크스페이스 파일 작성**

`package.json`:

```json
{
  "name": "weather-app",
  "private": true,
  "type": "module",
  "packageManager": "pnpm@11.22.0",
  "engines": {
    "node": ">=22.12"
  },
  "scripts": {
    "dev": "pnpm -r --parallel dev",
    "build": "pnpm --filter web build && pnpm --filter server build",
    "test": "pnpm -r test",
    "typecheck": "pnpm -r typecheck"
  }
}
```

`pnpm-workspace.yaml`:

```yaml
packages:
  - server
  - web
allowBuilds:
  esbuild: true
```

`tsconfig.base.json`:

```json
{
  "compilerOptions": {
    "target": "ES2023",
    "lib": ["ES2023"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "isolatedModules": true
  }
}
```

`.gitignore`에 다음 줄을 추가(이미 있는 줄은 그대로 둔다):

```
.pytest_cache/
*.tsbuildinfo
```

- [ ] **Step 2: server 패키지 작성과 의존성 설치**

`server/package.json`:

```json
{
  "name": "server",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc -p tsconfig.build.json",
    "start": "node dist/index.js",
    "test": "vitest run",
    "typecheck": "tsc -p tsconfig.json",
    "regions": "tsx scripts/build-regions.ts"
  }
}
```

`server/tsconfig.json`:

```json
{
  "extends": "../tsconfig.base.json",
  "compilerOptions": {
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "types": ["node"],
    "resolveJsonModule": true,
    "noEmit": true
  },
  "include": ["src", "test", "scripts"]
}
```

`server/tsconfig.build.json`:

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "noEmit": false,
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"]
}
```

Run:

```bash
pnpm --filter server add hono@^4.13.14 @hono/node-server@^2.1.5
pnpm --filter server add -D typescript@^5.9.3 tsx@^4.23.15 vitest@^5.0.5 @types/node@^24 exceljs@^4.4.0
```

Expected: `pnpm-lock.yaml` 생성, `server/node_modules` 생성. (web 폴더가 아직 없어도 workspace 패턴이라 오류가 아니다)

- [ ] **Step 3: 응답 타입 작성**

`server/src/types.ts`:

```ts
// /weather/api/forecast 응답 타입. web도 이 파일을 import 하므로 다른 모듈을 import 하지 않는다

export interface Location {
  lat: number
  lon: number
  nx: number
  ny: number
  /** "서울특별시 종로구 사직동" */
  name: string
}

export interface Current {
  /** 관측 기준 시각 (KST ISO) */
  observedAt: string
  temp: number | null
  humidity: number | null
  rain1h: number | null
  windSpeed: number | null
  windDir: number | null
  /** 강수형태 코드 (0 없음, 1 비, 2 비/눈, 3 눈, 5 빗방울, 6 빗방울눈날림, 7 눈날림) */
  pty: number | null
}

export interface Hourly {
  /** 예보 시각 (KST ISO) */
  time: string
  temp: number | null
  /** 하늘상태 코드 (1 맑음, 3 구름많음, 4 흐림) */
  sky: number | null
  /** 강수형태 코드 (0 없음, 1 비, 2 비/눈, 3 눈, 4 소나기) */
  pty: number | null
  pop: number | null
  humidity: number | null
  windDir: number | null
  /** 연장기간(정성 코드) 시각이면 null이고 windText에 문구가 들어간다 */
  windSpeed: number | null
  windText: string | null
  /** "1mm 미만", "2.0mm", "약한 비" 등. 강수 없으면 null */
  pcp: string | null
  /** "0.5cm 미만", "보통 눈" 등. 적설 없으면 null */
  sno: string | null
}

export interface Daily {
  /** YYYY-MM-DD (KST) */
  date: string
  min: number | null
  max: number | null
  amWf: string | null
  pmWf: string | null
  amPop: number | null
  pmPop: number | null
}

export interface ForecastResponse {
  /** 남한 밖이면 null (기상청을 호출하지 않는다) */
  location: Location | null
  current: Current | null
  hourly: Hourly[]
  daily: Daily[]
  /** 실제 사용한 발표시각 (UTC ISO) */
  issued: { current?: string; hourly?: string; mid?: string }
  /** 하나라도 만료된 캐시로 대신했으면 true */
  stale: boolean
  /** 섹션별 사용자용 오류 문구 */
  errors: { current?: string; hourly?: string; daily?: string }
}
```

- [ ] **Step 4: 실패하는 테스트 작성**

`server/test/app.test.ts`:

```ts
import { describe, expect, it } from 'vitest'
import { createApp } from '../src/app.js'

describe('app', () => {
  it('GET /weather/healthz 는 ok를 돌려준다', async () => {
    const res = await createApp({ forecast: null }).request('/weather/healthz')
    expect(res.status).toBe(200)
    expect(await res.json()).toEqual({ ok: true })
  })
})
```

- [ ] **Step 5: 테스트 실패 확인**

Run: `pnpm --filter server exec vitest run test/app.test.ts`
Expected: FAIL (`../src/app.js`를 찾을 수 없음)

- [ ] **Step 6: app과 진입점 구현**

`server/src/app.ts`:

```ts
import { Hono } from 'hono'
import type { ForecastResponse } from './types.js'

export interface ForecastProvider {
  get(lat: number, lon: number): Promise<ForecastResponse>
}

export interface AppDeps {
  /** KMA_SERVICE_KEY가 없으면 null (api는 503) */
  forecast: ForecastProvider | null
  /** web 빌드 결과 폴더 (절대 경로) */
  webDir?: string
  /** 바람 데이터 폴더 (절대 경로) */
  windDir?: string
}

export function createApp(deps: AppDeps) {
  const app = new Hono().basePath('/weather')

  app.get('/healthz', (c) => c.json({ ok: true }))

  return app
}
```

`server/src/index.ts`:

```ts
import { serve } from '@hono/node-server'
import { join } from 'node:path'
import { fileURLToPath } from 'node:url'
import { createApp } from './app.js'

// src/index.ts, dist/index.js 어느 쪽에서 실행해도 저장소 루트를 가리킨다
const ROOT = fileURLToPath(new URL('../../', import.meta.url))
// 이미 있는 환경변수(pm2 env)는 덮어쓰지 않는다
try {
  process.loadEnvFile(join(ROOT, '.env'))
} catch {
  // .env가 없으면 환경변수만 쓴다
}

const port = Number(process.env.PORT ?? 3090)
const app = createApp({ forecast: null })

serve({ fetch: app.fetch, port, hostname: '127.0.0.1' }, () => {
  console.log(`[weather] http://127.0.0.1:${port}/weather/`)
})
```

- [ ] **Step 7: 테스트·타입체크 통과 확인**

Run: `pnpm --filter server exec vitest run test/app.test.ts && pnpm --filter server typecheck`
Expected: 1 passed, 타입 오류 없음

- [ ] **Step 8: 커밋**

```bash
git add package.json pnpm-workspace.yaml pnpm-lock.yaml tsconfig.base.json .gitignore server/package.json server/tsconfig.json server/tsconfig.build.json server/src server/test/app.test.ts
git commit -m "feat: pnpm 워크스페이스와 서버 뼈대(/weather/healthz)" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: 위경도 → 단기예보 격자 변환

**Files:**
- Create: `server/src/kma/grid.ts`
- Test: `server/test/grid.test.ts`

**Interfaces:**
- Produces: `toGrid(lat: number, lon: number): GridPoint`, `isInGrid(p: GridPoint): boolean`, `interface GridPoint { nx: number; ny: number }`, `NX = 149`, `NY = 253`

- [ ] **Step 1: 실패하는 테스트 작성**

`server/test/grid.test.ts`:

```ts
import { describe, expect, it } from 'vitest'
import { isInGrid, toGrid } from '../src/kma/grid.js'

describe('toGrid', () => {
  it('가이드 예시 좌표를 (59, 125)로 바꾼다', () => {
    expect(toGrid(37.488201, 126.92981)).toEqual({ nx: 59, ny: 125 })
  })

  it('격자 엑셀의 서울특별시 행과 같다', () => {
    expect(toGrid(37.5635694444444, 126.980008333333)).toEqual({ nx: 60, ny: 127 })
  })
})

describe('isInGrid', () => {
  it('서울은 격자 안', () => {
    expect(isInGrid(toGrid(37.5636, 126.98))).toBe(true)
  })

  it('베이징·도쿄는 격자 밖', () => {
    expect(isInGrid(toGrid(39.9, 116.4))).toBe(false)
    expect(isInGrid(toGrid(35.68, 139.69))).toBe(false)
  })
})
```

- [ ] **Step 2: 테스트 실패 확인**

Run: `pnpm --filter server exec vitest run test/grid.test.ts`
Expected: FAIL (모듈 없음)

- [ ] **Step 3: 구현**

`server/src/kma/grid.ts`:

```ts
// 단기예보 격자 변환. 기상청 단기예보 가이드의 Lambert Conformal Conic C 예제를 옮겼다

const RE = 6371.00877 // 지구 반경 (km)
const GRID = 5.0 // 격자 간격 (km)
const SLAT1 = 30.0 // 표준위도 1
const SLAT2 = 60.0 // 표준위도 2
const OLON = 126.0 // 기준점 경도
const OLAT = 38.0 // 기준점 위도
const XO = 210 / GRID // 기준점 X 좌표 (격자)
const YO = 675 / GRID // 기준점 Y 좌표 (격자)

export const NX = 149
export const NY = 253

const DEGRAD = Math.PI / 180
const re = RE / GRID
const slat1 = SLAT1 * DEGRAD
const slat2 = SLAT2 * DEGRAD
const olon = OLON * DEGRAD
const olat = OLAT * DEGRAD
const sn =
  Math.log(Math.cos(slat1) / Math.cos(slat2)) /
  Math.log(Math.tan(Math.PI * 0.25 + slat2 * 0.5) / Math.tan(Math.PI * 0.25 + slat1 * 0.5))
const sf = (Math.pow(Math.tan(Math.PI * 0.25 + slat1 * 0.5), sn) * Math.cos(slat1)) / sn
const ro = (re * sf) / Math.pow(Math.tan(Math.PI * 0.25 + olat * 0.5), sn)

export interface GridPoint {
  nx: number
  ny: number
}

export function toGrid(lat: number, lon: number): GridPoint {
  const ra = (re * sf) / Math.pow(Math.tan(Math.PI * 0.25 + lat * DEGRAD * 0.5), sn)
  let theta = lon * DEGRAD - olon
  if (theta > Math.PI) theta -= 2 * Math.PI
  if (theta < -Math.PI) theta += 2 * Math.PI
  theta *= sn
  const x = ra * Math.sin(theta) + XO
  const y = ro - ra * Math.cos(theta) + YO
  // C 예제의 (int)(x + 1.5)
  return { nx: Math.floor(x + 1.5), ny: Math.floor(y + 1.5) }
}

export function isInGrid({ nx, ny }: GridPoint): boolean {
  return nx >= 1 && nx <= NX && ny >= 1 && ny <= NY
}
```

- [ ] **Step 4: 테스트 통과 확인**

Run: `pnpm --filter server exec vitest run test/grid.test.ts`
Expected: 4 passed

- [ ] **Step 5: 커밋**

```bash
git add server/src/kma/grid.ts server/test/grid.test.ts
git commit -m "feat: 위경도-단기예보 격자 변환" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: 발표시각 계산

**Files:**
- Create: `server/src/kma/baseTime.ts`
- Test: `server/test/baseTime.test.ts`

**Interfaces:**
- Produces:
  - `type IssueKind = 'ncst' | 'vilage' | 'mid'`
  - `latestIssue(kind: IssueKind, now: Date): Date`: now에 조회 가능한 가장 최근 발표
  - `previousIssue(kind: IssueKind, issue: Date): Date`: 바로 전 발표
  - `nextAvailable(kind: IssueKind, issue: Date): Date`: 다음 발표가 조회 가능해지는 시각
  - `formatBase(issue: Date): { baseDate: string; baseTime: string }`: KST `YYYYMMDD`, `HHmm`
  - `formatTmFc(issue: Date): string`: KST `YYYYMMDDHHmm`

규칙: ncst는 매시 정각 발표, vilage는 02·05·08·11·14·17·20·23시, mid는 06·18시. 셋 다 발표 10분 뒤부터 조회 가능.

- [ ] **Step 1: 실패하는 테스트 작성**

`server/test/baseTime.test.ts`:

```ts
import { describe, expect, it } from 'vitest'
import { formatBase, formatTmFc, latestIssue, nextAvailable, previousIssue } from '../src/kma/baseTime.js'

const kst = (s: string) => new Date(`${s}+09:00`)

describe('latestIssue', () => {
  it('초단기실황: 정시 10분 전에는 직전 시각', () => {
    expect(formatBase(latestIssue('ncst', kst('2026-10-27T15:09:00')))).toEqual({ baseDate: '20261027', baseTime: '1400' })
    expect(formatBase(latestIssue('ncst', kst('2026-10-27T15:10:00')))).toEqual({ baseDate: '20261027', baseTime: '1500' })
  })

  it('초단기실황: 자정 직후에는 전날 23시', () => {
    expect(formatBase(latestIssue('ncst', kst('2026-10-27T00:05:00')))).toEqual({ baseDate: '20261026', baseTime: '2300' })
  })

  it('단기예보: 3시간 간격 발표 + 10분', () => {
    expect(formatBase(latestIssue('vilage', kst('2026-10-27T14:09:00')))).toEqual({ baseDate: '20261027', baseTime: '1100' })
    expect(formatBase(latestIssue('vilage', kst('2026-10-27T14:10:00')))).toEqual({ baseDate: '20261027', baseTime: '1400' })
    expect(formatBase(latestIssue('vilage', kst('2026-10-27T02:05:00')))).toEqual({ baseDate: '20261026', baseTime: '2300' })
    expect(formatBase(latestIssue('vilage', kst('2026-10-27T01:00:00')))).toEqual({ baseDate: '20261026', baseTime: '2300' })
  })

  it('중기예보: 06시·18시 발표 + 10분', () => {
    expect(formatTmFc(latestIssue('mid', kst('2026-10-27T06:09:00')))).toBe('202610261800')
    expect(formatTmFc(latestIssue('mid', kst('2026-10-27T06:10:00')))).toBe('202610270600')
    expect(formatTmFc(latestIssue('mid', kst('2026-10-27T18:30:00')))).toBe('202610271800')
  })
})

describe('previousIssue', () => {
  it('바로 전 발표', () => {
    expect(formatBase(previousIssue('vilage', kst('2026-10-27T02:00:00')))).toEqual({ baseDate: '20261026', baseTime: '2300' })
    expect(formatTmFc(previousIssue('mid', kst('2026-10-27T06:00:00')))).toBe('202610261800')
    expect(formatBase(previousIssue('ncst', kst('2026-10-27T15:00:00')))).toEqual({ baseDate: '20261027', baseTime: '1400' })
  })
})

describe('nextAvailable', () => {
  it('다음 발표의 조회 가능 시각', () => {
    expect(nextAvailable('vilage', kst('2026-10-27T14:00:00'))).toEqual(kst('2026-10-27T17:10:00'))
    expect(nextAvailable('vilage', kst('2026-10-27T23:00:00'))).toEqual(kst('2026-10-28T02:10:00'))
    expect(nextAvailable('ncst', kst('2026-10-27T15:00:00'))).toEqual(kst('2026-10-27T16:10:00'))
    expect(nextAvailable('mid', kst('2026-10-27T06:00:00'))).toEqual(kst('2026-10-27T18:10:00'))
  })
})
```

- [ ] **Step 2: 테스트 실패 확인**

Run: `pnpm --filter server exec vitest run test/baseTime.test.ts`
Expected: FAIL (모듈 없음)

- [ ] **Step 3: 구현**

`server/src/kma/baseTime.ts`:

```ts
// 기상청 API별 발표시각 계산. 모든 시각은 KST 기준으로 판단한다

export type IssueKind = 'ncst' | 'vilage' | 'mid'

const HOUR = 3_600_000
const KST_OFFSET = 9 * HOUR

const SCHEDULE: Record<IssueKind, { hours: number[]; delayMin: number }> = {
  // 초단기실황: 매시 정각 발표, 10분 이후 제공
  ncst: { hours: Array.from({ length: 24 }, (_, h) => h), delayMin: 10 },
  // 단기예보: 1일 8회, 10분 이후 제공
  vilage: { hours: [2, 5, 8, 11, 14, 17, 20, 23], delayMin: 10 },
  // 중기예보: 06·18시 발표. 다른 API와 같이 10분 여유를 둔다
  mid: { hours: [6, 18], delayMin: 10 },
}

function kstHour(t: Date): number {
  return new Date(t.getTime() + KST_OFFSET).getUTCHours()
}

// KST 오프셋이 정수 시간이라 UTC 기준으로 내려도 KST 정시가 된다
function floorHour(t: Date): Date {
  return new Date(Math.floor(t.getTime() / HOUR) * HOUR)
}

function addHours(t: Date, hours: number): Date {
  return new Date(t.getTime() + hours * HOUR)
}

export function latestIssue(kind: IssueKind, now: Date): Date {
  const { hours, delayMin } = SCHEDULE[kind]
  for (let t = floorHour(now); ; t = addHours(t, -1)) {
    if (hours.includes(kstHour(t)) && now.getTime() >= t.getTime() + delayMin * 60_000) return t
  }
}

export function previousIssue(kind: IssueKind, issue: Date): Date {
  const { hours } = SCHEDULE[kind]
  for (let t = addHours(floorHour(issue), -1); ; t = addHours(t, -1)) {
    if (hours.includes(kstHour(t))) return t
  }
}

export function nextAvailable(kind: IssueKind, issue: Date): Date {
  const { hours, delayMin } = SCHEDULE[kind]
  for (let t = addHours(floorHour(issue), 1); ; t = addHours(t, 1)) {
    if (hours.includes(kstHour(t))) return new Date(t.getTime() + delayMin * 60_000)
  }
}

export function formatBase(issue: Date): { baseDate: string; baseTime: string } {
  const k = new Date(issue.getTime() + KST_OFFSET)
  const p = (n: number) => String(n).padStart(2, '0')
  return {
    baseDate: `${k.getUTCFullYear()}${p(k.getUTCMonth() + 1)}${p(k.getUTCDate())}`,
    baseTime: `${p(k.getUTCHours())}${p(k.getUTCMinutes())}`,
  }
}

export function formatTmFc(issue: Date): string {
  const { baseDate, baseTime } = formatBase(issue)
  return baseDate + baseTime
}
```

- [ ] **Step 4: 테스트 통과 확인**

Run: `pnpm --filter server exec vitest run test/baseTime.test.ts`
Expected: 6 passed

- [ ] **Step 5: 커밋**

```bash
git add server/src/kma/baseTime.ts server/test/baseTime.test.ts
git commit -m "feat: 기상청 발표시각 계산" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: 지역 데이터와 중기 구역코드 매핑

**Files:**
- Create: `server/scripts/build-regions.ts`
- Create (생성물): `server/src/data/towns.json`, `server/src/data/midTempRegions.json`
- Create: `server/src/regions.ts`
- Test: `server/test/regions.test.ts`

**Interfaces:**
- Produces:
  - `interface Town { sido: string; sigungu: string; town: string; lat: number; lon: number }`
  - `findNearestTown(lat: number, lon: number): { town: Town; distanceKm: number }`
  - `townName(t: Town): string`: 비어 있지 않은 이름을 공백으로 이은 것
  - `midLandRegId(t: Town): string | null`
  - `midTempRegId(t: Town, landRegId: string): string`

엑셀 열 (격자): 0 구분, 1 행정구역코드, 2 1단계(시도), 3 2단계(시군구), 4 3단계(읍면동), 5 격자X, 6 격자Y, 13 경도(초/100), 14 위도(초/100). `이어도` 행은 좌표가 0이라 제외한다.
엑셀 열 (중기): 0 예보구역코드, 1 구역명, 2 특성(`C`=도시: 중기기온 조회용).
주의: 2026년 개편으로 광주·전남은 `전남광주통합특별시` 하나다.

- [ ] **Step 1: 변환 스크립트 작성과 실행**

`server/scripts/build-regions.ts`:

```ts
// 기상청 가이드 엑셀(data-src/) → server/src/data/*.json
// 엑셀이 갱신되면 파일을 바꾸고 `pnpm --filter server regions`로 다시 만든다
import ExcelJS from 'exceljs'
import { mkdir, writeFile } from 'node:fs/promises'
import { fileURLToPath } from 'node:url'

const ROOT = fileURLToPath(new URL('../../', import.meta.url))
const OUT_DIR = fileURLToPath(new URL('../src/data/', import.meta.url))

async function readRows(file: string): Promise<string[][]> {
  const workbook = new ExcelJS.Workbook()
  await workbook.xlsx.readFile(`${ROOT}data-src/${file}`)
  const sheet = workbook.worksheets[0]
  if (!sheet) throw new Error(`${file}: 시트가 없습니다`)
  const rows: string[][] = []
  sheet.eachRow((row) => {
    // row.values는 1부터 시작하는 희소 배열이라 Array.from으로 빈칸을 undefined로 채운다
    rows.push(Array.from(row.values as unknown[]).slice(1).map((v) => (v == null ? '' : String(v).trim())))
  })
  return rows.slice(1) // 머리글 제외
}

const round6 = (n: number) => Math.round(n * 1e6) / 1e6

const towns = (await readRows('kma-grid-latlon-2610.xlsx'))
  .map((r) => ({
    sido: r[2] ?? '',
    sigungu: r[3] ?? '',
    town: r[4] ?? '',
    lat: round6(Number(r[14])),
    lon: round6(Number(r[13])),
  }))
  .filter((t) => t.sido && t.lat > 0 && t.lon > 0)

const midTempRegions = (await readRows('kma-mid-temp-regions-2512.xlsx'))
  .filter((r) => r[2] === 'C')
  .map((r) => ({ code: r[0] ?? '', name: r[1] ?? '' }))

await mkdir(OUT_DIR, { recursive: true })
await writeFile(`${OUT_DIR}towns.json`, JSON.stringify(towns))
await writeFile(`${OUT_DIR}midTempRegions.json`, JSON.stringify(midTempRegions))
console.log(`towns ${towns.length}개, midTempRegions ${midTempRegions.length}개`)
```

Run: `pnpm --filter server regions`
Expected: `towns 3837개, midTempRegions 2xx개` 비슷한 출력 (towns는 이어도 2행을 뺀 3,837개 근처). 그 뒤 확인:

Run: `node -e "const t=require('./server/src/data/towns.json');console.log(t[0], t.filter(x=>x.sido==='전남광주통합특별시').length)"`
Expected: `{ sido: '서울특별시', sigungu: '', town: '', lat: 37.563569, lon: 126.980008 } 421` 비슷한 값

- [ ] **Step 2: 실패하는 테스트 작성**

`server/test/regions.test.ts`:

```ts
import { describe, expect, it } from 'vitest'
import { findNearestTown, midLandRegId, midTempRegId, townName, type Town } from '../src/regions.js'

const t = (sido: string, sigungu = '', town = ''): Town => ({ sido, sigungu, town, lat: 0, lon: 0 })

describe('findNearestTown', () => {
  it('서울시청 근처는 서울특별시', () => {
    const { town, distanceKm } = findNearestTown(37.5636, 126.98)
    expect(town.sido).toBe('서울특별시')
    expect(distanceKm).toBeLessThan(2)
  })

  it('황해 한가운데는 20km보다 멀다', () => {
    expect(findNearestTown(35, 120).distanceKm).toBeGreaterThan(20)
  })
})

describe('townName', () => {
  it('빈 이름은 건너뛴다', () => {
    expect(townName(t('서울특별시', '종로구', '사직동'))).toBe('서울특별시 종로구 사직동')
    expect(townName(t('서울특별시'))).toBe('서울특별시')
  })
})

describe('midLandRegId', () => {
  it('시도별 육상 구역', () => {
    expect(midLandRegId(t('서울특별시', '종로구'))).toBe('11B00000')
    expect(midLandRegId(t('경기도', '수원시장안구'))).toBe('11B00000')
    expect(midLandRegId(t('제주특별자치도', '제주시'))).toBe('11G00000')
    expect(midLandRegId(t('전남광주통합특별시', '광산구'))).toBe('11F20000')
    expect(midLandRegId(t('세종특별자치시', '세종특별자치시'))).toBe('11C20000')
  })

  it('강원은 영서/영동을 나눈다', () => {
    expect(midLandRegId(t('강원특별자치도', '강릉시'))).toBe('11D20000')
    expect(midLandRegId(t('강원특별자치도', '춘천시'))).toBe('11D10000')
  })
})

describe('midTempRegId', () => {
  const temp = (town: Town) => midTempRegId(town, midLandRegId(town)!)

  it('광역시의 구는 시 대표 코드', () => {
    expect(temp(t('서울특별시', '종로구'))).toBe('11B10101')
    expect(temp(t('전남광주통합특별시', '광산구'))).toBe('11F20501')
    expect(temp(t('부산광역시', '기장군'))).toBe('11H20201')
  })

  it('시군 이름으로 찾는다 (구가 붙은 시 포함)', () => {
    expect(temp(t('경기도', '수원시장안구'))).toBe('11B20601')
    expect(temp(t('제주특별자치도', '서귀포시'))).toBe('11G00401')
    expect(temp(t('세종특별자치시', '세종특별자치시'))).toBe('11C20404')
  })

  it('같은 이름은 육상 구역 안에서 고른다', () => {
    expect(temp(t('강원특별자치도', '고성군'))).toBe('11D20402')
    expect(temp(t('경상남도', '고성군'))).toBe('11H20404')
    expect(temp(t('경기도', '광주시'))).toBe('11B20702')
  })

  it('못 찾으면 구역 대표 도시', () => {
    expect(temp(t('경상북도', '울릉군'))).toBe('11H10701')
  })
})
```

- [ ] **Step 3: 테스트 실패 확인**

Run: `pnpm --filter server exec vitest run test/regions.test.ts`
Expected: FAIL (모듈 없음)

- [ ] **Step 4: 구현**

`server/src/regions.ts`:

```ts
// 위치 → 가까운 읍면동 → 중기예보 육상/기온 구역코드
import midTempRegions from './data/midTempRegions.json' with { type: 'json' }
import towns from './data/towns.json' with { type: 'json' }

export interface Town {
  sido: string
  sigungu: string
  town: string
  lat: number
  lon: number
}

const TOWNS: Town[] = towns
const EARTH_RADIUS_KM = 6371

function distanceKm(lat1: number, lon1: number, lat2: number, lon2: number): number {
  const rad = Math.PI / 180
  const dLat = (lat2 - lat1) * rad
  const dLon = (lon2 - lon1) * rad
  const a = Math.sin(dLat / 2) ** 2 + Math.cos(lat1 * rad) * Math.cos(lat2 * rad) * Math.sin(dLon / 2) ** 2
  return 2 * EARTH_RADIUS_KM * Math.asin(Math.sqrt(a))
}

export function findNearestTown(lat: number, lon: number): { town: Town; distanceKm: number } {
  let best = TOWNS[0]!
  let bestDistance = Infinity
  for (const town of TOWNS) {
    const d = distanceKm(lat, lon, town.lat, town.lon)
    if (d < bestDistance) {
      best = town
      bestDistance = d
    }
  }
  return { town: best, distanceKm: bestDistance }
}

export function townName(t: Town): string {
  return [t.sido, t.sigungu, t.town].filter(Boolean).join(' ')
}

const SIDO_LAND: Record<string, string> = {
  서울특별시: '11B00000',
  인천광역시: '11B00000',
  경기도: '11B00000',
  대전광역시: '11C20000',
  세종특별자치시: '11C20000',
  충청남도: '11C20000',
  충청북도: '11C10000',
  전남광주통합특별시: '11F20000',
  광주광역시: '11F20000',
  전라남도: '11F20000',
  전북특별자치도: '11F10000',
  대구광역시: '11H10000',
  경상북도: '11H10000',
  부산광역시: '11H20000',
  울산광역시: '11H20000',
  경상남도: '11H20000',
  제주특별자치도: '11G00000',
}

// 중기 구역 "강원영동"에 속하는 시군 (나머지 강원은 영서)
const GANGWON_YEONGDONG = new Set(['강릉시', '동해시', '삼척시', '속초시', '고성군', '양양군', '태백시'])

export function midLandRegId(t: Town): string | null {
  if (t.sido.startsWith('강원')) return GANGWON_YEONGDONG.has(t.sigungu) ? '11D20000' : '11D10000'
  return SIDO_LAND[t.sido] ?? null
}

// 육상 구역별로 고를 수 있는 기온 코드 접두사와, 못 찾았을 때 쓰는 대표 도시
const LAND_FAMILY: Record<string, { prefixes: string[]; fallback: string }> = {
  '11B00000': { prefixes: ['11B', '11A'], fallback: '11B10101' }, // 서울
  '11D10000': { prefixes: ['11D1'], fallback: '11D10301' }, // 춘천
  '11D20000': { prefixes: ['11D2'], fallback: '11D20501' }, // 강릉
  '11C20000': { prefixes: ['11C2'], fallback: '11C20401' }, // 대전
  '11C10000': { prefixes: ['11C1'], fallback: '11C10301' }, // 청주
  '11F20000': { prefixes: ['11F2', '21F2'], fallback: '11F20501' }, // 광주
  '11F10000': { prefixes: ['11F1', '21F1'], fallback: '11F10201' }, // 전주
  '11H10000': { prefixes: ['11H1', '11E'], fallback: '11H10701' }, // 대구
  '11H20000': { prefixes: ['11H2'], fallback: '11H20201' }, // 부산
  '11G00000': { prefixes: ['11G'], fallback: '11G00201' }, // 제주
}

// 시군구가 "종로구"처럼 구뿐인 광역시는 시 이름으로 찾는다
const METRO_CITY: Record<string, string> = {
  서울특별시: '서울',
  인천광역시: '인천',
  대전광역시: '대전',
  대구광역시: '대구',
  부산광역시: '부산',
  울산광역시: '울산',
  세종특별자치시: '세종',
  전남광주통합특별시: '광주',
  광주광역시: '광주',
}

export function midTempRegId(t: Town, landRegId: string): string {
  const family = LAND_FAMILY[landRegId]
  if (!family) return LAND_FAMILY['11B00000']!.fallback
  const candidates = midTempRegions.filter((r) => family.prefixes.some((p) => r.code.startsWith(p)))
  // "수원시장안구" → "수원", "군위군" → "군위"
  const cityName = t.sigungu.match(/^(.+?)(시|군)/)?.[1]
  const names = [t.sigungu, cityName, METRO_CITY[t.sido]].filter((n): n is string => Boolean(n))
  for (const name of names) {
    const hit = candidates.find((r) => r.name === name)
    if (hit) return hit.code
  }
  return family.fallback
}
```

- [ ] **Step 5: 테스트·타입체크 통과 확인**

Run: `pnpm --filter server exec vitest run test/regions.test.ts && pnpm --filter server typecheck`
Expected: 모두 통과. 실패하는 기댓값이 있으면 `server/src/data/midTempRegions.json`에서 해당 이름·코드를 확인해 매핑 규칙을 고친다(테스트 기댓값은 가이드 엑셀 기준이라 바꾸지 않는다).

- [ ] **Step 6: 커밋**

```bash
git add server/scripts/build-regions.ts server/src/data server/src/regions.ts server/test/regions.test.ts data-src
git commit -m "feat: 가까운 읍면동과 중기 육상/기온 구역코드 매핑" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: 기상청 클라이언트와 TTL 캐시

**Files:**
- Create: `server/src/kma/client.ts`, `server/src/cache.ts`
- Test: `server/test/client.test.ts`, `server/test/cache.test.ts`

**Interfaces:**
- Produces:
  - `class KmaError extends Error { readonly code: string }`: code는 기상청 코드(`'03'` 데이터없음, `'22'` 한도초과, `'20'`·`'30'` 인증 등) 또는 `'NETWORK'`, `'PARSE'`
  - `interface KmaClient { get<T>(path: string, params: Record<string, string | number>): Promise<T[]> }`: `path`는 `'VilageFcstInfoService_2.0/getUltraSrtNcst'`처럼 `1360000/` 뒤 부분
  - `createKmaClient(opts: { serviceKey: string; fetch?: typeof fetch; timeoutMs?: number; baseUrl?: string }): KmaClient`
  - `parseKmaBody<T>(text: string): T[]`
  - `class TtlCache<T> { constructor(maxEntries?: number, now?: () => number); getOrLoad(key: string, load: () => Promise<Loaded<T>>): Promise<CacheResult<T>>; readonly size: number }`
  - `interface Loaded<T> { value: T; expiresAt: number }`, `interface CacheResult<T> { value: T; stale: boolean }`

- [ ] **Step 1: 클라이언트 실패 테스트 작성**

`server/test/client.test.ts`:

```ts
import { readFileSync } from 'node:fs'
import { describe, expect, it } from 'vitest'
import { createKmaClient, KmaError, parseKmaBody } from '../src/kma/client.js'

const fixture = (name: string) => readFileSync(new URL(`./fixtures/${name}`, import.meta.url), 'utf8')

describe('parseKmaBody', () => {
  it('정상 응답의 item 배열을 돌려준다', () => {
    const items = parseKmaBody<{ category: string }>(fixture('ncst.json'))
    expect(items).toHaveLength(8)
    expect(items[0]?.category).toBe('PTY')
  })

  it('resultCode가 00이 아니면 KmaError', () => {
    const body = JSON.stringify({ response: { header: { resultCode: '03', resultMsg: 'NO_DATA' } } })
    expect(() => parseKmaBody(body)).toThrow(KmaError)
    try {
      parseKmaBody(body)
    } catch (e) {
      expect((e as KmaError).code).toBe('03')
    }
  })

  it('인증 오류 봉투(JSON)를 해석한다', () => {
    const body = JSON.stringify({
      OpenAPI_ServiceResponse: { cmmMsgHeader: { errMsg: 'SERVICE_KEY_IS_NULL', returnAuthMsg: '서비스 접근거부', returnReasonCode: '20' } },
    })
    expect(() => parseKmaBody(body)).toThrow(expect.objectContaining({ code: '20' }))
  })

  it('XML 오류 봉투를 해석한다', () => {
    const body =
      '<OpenAPI_ServiceResponse><cmmMsgHeader><errMsg>SERVICE ERROR</errMsg><returnAuthMsg>LIMITED_NUMBER_OF_SERVICE_REQUESTS_EXCEEDS_ERROR</returnAuthMsg><returnReasonCode>22</returnReasonCode></cmmMsgHeader></OpenAPI_ServiceResponse>'
    expect(() => parseKmaBody(body)).toThrow(expect.objectContaining({ code: '22' }))
  })
})

describe('createKmaClient', () => {
  function recordingFetch(body: string) {
    const urls: string[] = []
    const fetchFn = (async (url: string | URL | Request) => {
      urls.push(String(url))
      return new Response(body)
    }) as typeof fetch
    return { urls, fetchFn }
  }

  it('Decoding 키는 URL 인코딩하고 파라미터를 붙인다', async () => {
    const { urls, fetchFn } = recordingFetch(fixture('ncst.json'))
    const kma = createKmaClient({ serviceKey: 'ab+c/d==', fetch: fetchFn })
    await kma.get('VilageFcstInfoService_2.0/getUltraSrtNcst', { base_date: '20261027', nx: 60 })
    expect(urls[0]).toContain('https://apis.data.go.kr/1360000/VilageFcstInfoService_2.0/getUltraSrtNcst?')
    expect(urls[0]).toContain('serviceKey=ab%2Bc%2Fd%3D%3D&')
    expect(urls[0]).toContain('dataType=JSON')
    expect(urls[0]).toContain('base_date=20261027')
    expect(urls[0]).toContain('nx=60')
  })

  it('이미 인코딩된 Encoding 키는 그대로 쓴다', async () => {
    const { urls, fetchFn } = recordingFetch(fixture('ncst.json'))
    const kma = createKmaClient({ serviceKey: 'ab%2Bc%3D%3D', fetch: fetchFn })
    await kma.get('X/y', {})
    expect(urls[0]).toContain('serviceKey=ab%2Bc%3D%3D&')
  })

  it('네트워크 오류는 NETWORK 코드', async () => {
    const failing = (async () => {
      throw new TypeError('fetch failed')
    }) as typeof fetch
    const kma = createKmaClient({ serviceKey: 'k', fetch: failing })
    await expect(kma.get('X/y', {})).rejects.toMatchObject({ code: 'NETWORK' })
  })
})
```

- [ ] **Step 2: 테스트 실패 확인**

Run: `pnpm --filter server exec vitest run test/client.test.ts`
Expected: FAIL (모듈 없음)

- [ ] **Step 3: 클라이언트 구현**

`server/src/kma/client.ts`:

```ts
// 공공데이터포털 기상청 API 호출과 응답 봉투 해석

const BASE_URL = 'https://apis.data.go.kr/1360000'

export class KmaError extends Error {
  constructor(
    readonly code: string,
    message: string,
  ) {
    super(message)
    this.name = 'KmaError'
  }
}

export interface KmaClient {
  get<T>(path: string, params: Record<string, string | number>): Promise<T[]>
}

export interface KmaClientOptions {
  serviceKey: string
  fetch?: typeof fetch
  timeoutMs?: number
  baseUrl?: string
}

export function createKmaClient(opts: KmaClientOptions): KmaClient {
  const doFetch = opts.fetch ?? fetch
  // 포털의 Encoding 키(이미 %인코딩됨)와 Decoding 키를 모두 받는다
  const key = opts.serviceKey.includes('%') ? opts.serviceKey : encodeURIComponent(opts.serviceKey)
  const baseUrl = opts.baseUrl ?? BASE_URL

  return {
    async get<T>(path: string, params: Record<string, string | number>): Promise<T[]> {
      const query = new URLSearchParams({ dataType: 'JSON', pageNo: '1' })
      for (const [k, v] of Object.entries(params)) query.set(k, String(v))
      // serviceKey는 URLSearchParams에 넣으면 %가 다시 인코딩되므로 직접 붙인다
      const url = `${baseUrl}/${path}?serviceKey=${key}&${query}`
      let text: string
      try {
        const res = await doFetch(url, { signal: AbortSignal.timeout(opts.timeoutMs ?? 10_000) })
        text = await res.text()
      } catch (err) {
        throw new KmaError('NETWORK', `기상청 요청 실패: ${(err as Error).message}`)
      }
      return parseKmaBody<T>(text)
    },
  }
}

export function parseKmaBody<T>(text: string): T[] {
  let json: any
  try {
    json = JSON.parse(text)
  } catch {
    // 인증·한도 오류는 dataType과 상관없이 XML로 올 때가 있다
    const tag = (name: string) => text.match(new RegExp(`<${name}>([^<]*)</${name}>`))?.[1]
    throw new KmaError(
      tag('returnReasonCode') ?? tag('resultCode') ?? 'PARSE',
      tag('returnAuthMsg') ?? tag('resultMsg') ?? '응답을 해석할 수 없습니다',
    )
  }
  const authHeader = json?.OpenAPI_ServiceResponse?.cmmMsgHeader
  if (authHeader) {
    throw new KmaError(String(authHeader.returnReasonCode ?? 'AUTH'), String(authHeader.returnAuthMsg ?? authHeader.errMsg ?? '서비스 오류'))
  }
  const code = String(json?.response?.header?.resultCode ?? 'PARSE')
  if (code !== '00') throw new KmaError(code, String(json?.response?.header?.resultMsg ?? '알 수 없는 오류'))
  const item = json?.response?.body?.items?.item
  if (Array.isArray(item)) return item as T[]
  return item ? [item as T] : []
}
```

- [ ] **Step 4: 클라이언트 테스트 통과 확인**

Run: `pnpm --filter server exec vitest run test/client.test.ts`
Expected: 7 passed

- [ ] **Step 5: 캐시 실패 테스트 작성**

`server/test/cache.test.ts`:

```ts
import { describe, expect, it, vi } from 'vitest'
import { TtlCache } from '../src/cache.js'

describe('TtlCache', () => {
  it('만료 전에는 다시 불러오지 않는다', async () => {
    let now = 0
    const cache = new TtlCache<string>(10, () => now)
    const load = vi.fn(async () => ({ value: 'a', expiresAt: 100 }))
    expect(await cache.getOrLoad('k', load)).toEqual({ value: 'a', stale: false })
    now = 99
    expect(await cache.getOrLoad('k', load)).toEqual({ value: 'a', stale: false })
    expect(load).toHaveBeenCalledTimes(1)
  })

  it('만료 후에는 다시 불러온다', async () => {
    let now = 0
    const cache = new TtlCache<string>(10, () => now)
    await cache.getOrLoad('k', async () => ({ value: 'a', expiresAt: 100 }))
    now = 100
    expect(await cache.getOrLoad('k', async () => ({ value: 'b', expiresAt: 200 }))).toEqual({ value: 'b', stale: false })
  })

  it('다시 불러오다 실패하면 이전 값을 stale로 준다', async () => {
    let now = 0
    const cache = new TtlCache<string>(10, () => now)
    await cache.getOrLoad('k', async () => ({ value: 'a', expiresAt: 100 }))
    now = 150
    const result = await cache.getOrLoad('k', async () => {
      throw new Error('down')
    })
    expect(result).toEqual({ value: 'a', stale: true })
  })

  it('이전 값이 없으면 오류를 그대로 던진다', async () => {
    const cache = new TtlCache<string>()
    await expect(
      cache.getOrLoad('k', async () => {
        throw new Error('down')
      }),
    ).rejects.toThrow('down')
  })

  it('동시에 들어온 요청은 한 번만 불러온다', async () => {
    const cache = new TtlCache<string>()
    let release: (v: { value: string; expiresAt: number }) => void = () => {}
    const load = vi.fn(() => new Promise<{ value: string; expiresAt: number }>((r) => (release = r)))
    const a = cache.getOrLoad('k', load)
    const b = cache.getOrLoad('k', load)
    release({ value: 'x', expiresAt: Date.now() + 1000 })
    expect(await a).toEqual({ value: 'x', stale: false })
    expect(await b).toEqual({ value: 'x', stale: false })
    expect(load).toHaveBeenCalledTimes(1)
  })

  it('최대 개수를 넘으면 가장 오래 안 쓴 키를 버린다', async () => {
    const cache = new TtlCache<string>(2, () => 0)
    const load = (v: string) => async () => ({ value: v, expiresAt: 100 })
    await cache.getOrLoad('a', load('a'))
    await cache.getOrLoad('b', load('b'))
    await cache.getOrLoad('a', load('a2')) // a를 최근 사용으로
    await cache.getOrLoad('c', load('c')) // b가 밀려난다
    expect(cache.size).toBe(2)
    expect(await cache.getOrLoad('b', load('b2'))).toEqual({ value: 'b2', stale: false })
  })
})
```

- [ ] **Step 6: 캐시 테스트 실패 확인**

Run: `pnpm --filter server exec vitest run test/cache.test.ts`
Expected: FAIL (모듈 없음)

- [ ] **Step 7: 캐시 구현**

`server/src/cache.ts`:

```ts
// 키별 TTL 캐시. 만료 뒤 다시 불러오다 실패하면 이전 값을 stale로 돌려준다.
// 같은 키로 동시에 들어온 요청은 한 번의 로드를 공유한다

export interface Loaded<T> {
  value: T
  /** epoch ms */
  expiresAt: number
}

export interface CacheResult<T> {
  value: T
  stale: boolean
}

export class TtlCache<T> {
  private readonly entries = new Map<string, Loaded<T>>()
  private readonly inflight = new Map<string, Promise<T>>()

  constructor(
    private readonly maxEntries = 5000,
    private readonly now: () => number = Date.now,
  ) {}

  get size(): number {
    return this.entries.size
  }

  async getOrLoad(key: string, load: () => Promise<Loaded<T>>): Promise<CacheResult<T>> {
    const hit = this.entries.get(key)
    if (hit && hit.expiresAt > this.now()) {
      this.remember(key, hit)
      return { value: hit.value, stale: false }
    }
    let pending = this.inflight.get(key)
    if (!pending) {
      pending = load()
        .then((loaded) => {
          this.remember(key, loaded)
          return loaded.value
        })
        .finally(() => this.inflight.delete(key))
      this.inflight.set(key, pending)
    }
    try {
      return { value: await pending, stale: false }
    } catch (err) {
      if (hit) return { value: hit.value, stale: true }
      throw err
    }
  }

  // Map은 넣은 순서를 지키므로 지웠다 다시 넣으면 가장 최근 사용이 된다
  private remember(key: string, entry: Loaded<T>): void {
    this.entries.delete(key)
    this.entries.set(key, entry)
    if (this.entries.size > this.maxEntries) {
      const oldest = this.entries.keys().next().value
      if (oldest !== undefined) this.entries.delete(oldest)
    }
  }
}
```

- [ ] **Step 8: 테스트 통과 확인**

Run: `pnpm --filter server exec vitest run test/client.test.ts test/cache.test.ts && pnpm --filter server typecheck`
Expected: 13 passed, 타입 오류 없음

- [ ] **Step 9: 커밋**

```bash
git add server/src/kma/client.ts server/src/cache.ts server/test/client.test.ts server/test/cache.test.ts server/test/fixtures
git commit -m "feat: 기상청 API 클라이언트와 TTL 캐시" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: 응답 파싱 (현재·시간별·10일)

**Files:**
- Create: `server/src/kma/parse.ts`
- Test: `server/test/parse.test.ts`

**Interfaces:**
- Consumes: `Current`, `Hourly`, `Daily` (`server/src/types.ts`)
- Produces:
  - `interface NcstItem { baseDate: string; baseTime: string; category: string; obsrValue: string; nx: number; ny: number }`
  - `interface FcstItem { baseDate: string; baseTime: string; category: string; fcstDate: string; fcstTime: string; fcstValue: string; nx: number; ny: number }`
  - `type MidItem = Record<string, string | number>`
  - `interface MidData { tmFc: string; land: MidItem | null; ta: MidItem | null }`: tmFc는 `YYYYMMDDHHmm`
  - `num(v: unknown): number | null`, `kstIso(date: string, time?: string): string`, `wfText(sky: number | null, pty: number | null): string | null`
  - `parseCurrent(items: NcstItem[]): Current | null`, `parseHourly(items: FcstItem[]): Hourly[]`, `buildDaily(fcst: FcstItem[], mid: MidData | null): Daily[]`

fixture 값(2026-10-27 서울, 단기 14시 발표): 66개 시각, 첫 시각 15시 TMP 14·SKY 1·REH 30·VEC 305·WSD 5.4. 10/28 TMN 2·TMX 13, 10/29 TMN 5·TMX 14(09시 맑음, 15시 SKY 4·PTY 1, 오전 POP 최대 30, 오후 60), 10/30 TMN 10·TMX 19, 10/31은 00시 한 칸(PCP `0`, WSD `1` → 정성 코드). 10/27의 TMP는 7~14. 중기 06시: `taMin4` 5, `taMax4` 13, `wf4Am`·`wf4Pm` 맑음, `rnSt4Am`·`rnSt4Pm` 10, `taMin8` 3, `taMax8` 13, `wf8` 맑음, `rnSt8` 10.

- [ ] **Step 1: 실패하는 테스트 작성**

`server/test/parse.test.ts`:

```ts
import { readFileSync } from 'node:fs'
import { describe, expect, it } from 'vitest'
import { parseKmaBody } from '../src/kma/client.js'
import { buildDaily, num, parseCurrent, parseHourly, wfText, type FcstItem, type MidItem, type NcstItem } from '../src/kma/parse.js'

const items = <T>(name: string) => parseKmaBody<T>(readFileSync(new URL(`./fixtures/${name}`, import.meta.url), 'utf8'))
const ncst = items<NcstItem>('ncst.json')
const vilage = items<FcstItem>('vilage.json')
const mid = { tmFc: '202610270600', land: items<MidItem>('midLand.json')[0]!, ta: items<MidItem>('midTa.json')[0]! }

describe('num', () => {
  it('결측과 숫자가 아닌 값은 null', () => {
    expect(num('1.5')).toBe(1.5)
    expect(num(13)).toBe(13)
    expect(num('-999')).toBeNull()
    expect(num('900')).toBeNull()
    expect(num('abc')).toBeNull()
    expect(num('')).toBeNull()
    expect(num(undefined)).toBeNull()
  })
})

describe('parseCurrent', () => {
  it('초단기실황을 현재 날씨로', () => {
    expect(parseCurrent(ncst)).toEqual({
      observedAt: '2026-10-27T15:00:00+09:00',
      temp: 15.3,
      humidity: 25,
      rain1h: 0,
      windSpeed: 4.6,
      windDir: 285,
      pty: 0,
    })
  })

  it('항목이 없으면 null', () => {
    expect(parseCurrent([])).toBeNull()
  })
})

describe('parseHourly', () => {
  const hourly = parseHourly(vilage)
  const at = (time: string) => hourly.find((h) => h.time === time)

  it('시각별로 묶는다', () => {
    expect(hourly).toHaveLength(66)
    expect(hourly[0]).toEqual({
      time: '2026-10-27T15:00:00+09:00',
      temp: 14,
      sky: 1,
      pty: 0,
      pop: 0,
      humidity: 30,
      windDir: 305,
      windSpeed: 5.4,
      windText: null,
      pcp: null,
      sno: null,
    })
  })

  it('정량 강수량 문자열은 그대로', () => {
    expect(at('2026-10-29T14:00:00+09:00')?.pcp).toBe('1mm 미만')
    expect(at('2026-10-29T16:00:00+09:00')?.pcp).toBe('1.0mm')
  })

  it('PCP가 숫자 코드인 시각은 정성 정보로 바꾼다', () => {
    expect(at('2026-10-30T00:00:00+09:00')).toMatchObject({ windSpeed: 1.1, windText: null })
    expect(at('2026-10-31T00:00:00+09:00')).toMatchObject({ windSpeed: null, windText: '약한 바람', pcp: null, sno: null })
  })

  it('코드값 문구', () => {
    const base = { baseDate: '20261027', baseTime: '1400', fcstDate: '20261031', fcstTime: '0300', nx: 60, ny: 127 }
    const coded = parseHourly([
      { ...base, category: 'PCP', fcstValue: '2' },
      { ...base, category: 'SNO', fcstValue: '1' },
      { ...base, category: 'WSD', fcstValue: '3' },
    ])
    expect(coded[0]).toMatchObject({ pcp: '보통 비', sno: '보통 눈', windText: '강한 바람', windSpeed: null })
  })
})

describe('wfText', () => {
  it('하늘상태와 강수형태를 중기예보 문구처럼', () => {
    expect(wfText(1, 0)).toBe('맑음')
    expect(wfText(3, 0)).toBe('구름많음')
    expect(wfText(4, 1)).toBe('흐리고 비')
    expect(wfText(3, 4)).toBe('구름많고 소나기')
    expect(wfText(null, null)).toBeNull()
  })
})

describe('buildDaily', () => {
  const daily = buildDaily(vilage, mid)
  const day = (date: string) => daily.find((d) => d.date === date)

  it('단기(발표일·TMN/TMX 있는 날) + 중기(+4~+10일)로 11일', () => {
    expect(daily.map((d) => d.date)).toEqual([
      '2026-10-27', '2026-10-28', '2026-10-29', '2026-10-30', '2026-10-31',
      '2026-11-01', '2026-11-02', '2026-11-03', '2026-11-04', '2026-11-05', '2026-11-06',
    ])
  })

  it('발표일에 TMN/TMX가 없으면 남은 시간 TMP로', () => {
    expect(day('2026-10-27')).toEqual({ date: '2026-10-27', min: 7, max: 14, amWf: null, pmWf: '맑음', amPop: null, pmPop: 0 })
  })

  it('단기예보 날', () => {
    expect(day('2026-10-28')).toEqual({ date: '2026-10-28', min: 2, max: 13, amWf: '맑음', pmWf: '맑음', amPop: 0, pmPop: 0 })
    expect(day('2026-10-29')).toEqual({ date: '2026-10-29', min: 5, max: 14, amWf: '맑음', pmWf: '흐리고 비', amPop: 30, pmPop: 60 })
  })

  it('중기예보 날 (오전/오후 구분은 7일까지)', () => {
    expect(day('2026-10-31')).toEqual({ date: '2026-10-31', min: 5, max: 13, amWf: '맑음', pmWf: '맑음', amPop: 10, pmPop: 10 })
    expect(day('2026-11-04')).toEqual({ date: '2026-11-04', min: 3, max: 13, amWf: '맑음', pmWf: '맑음', amPop: 10, pmPop: 10 })
  })

  it('단기예보가 없으면 중기만', () => {
    const onlyMid = buildDaily([], mid)
    expect(onlyMid).toHaveLength(7)
    expect(onlyMid[0]?.date).toBe('2026-10-31')
  })
})
```

- [ ] **Step 2: 테스트 실패 확인**

Run: `pnpm --filter server exec vitest run test/parse.test.ts`
Expected: FAIL (모듈 없음)

- [ ] **Step 3: 구현**

`server/src/kma/parse.ts`:

```ts
// 기상청 응답 → API 응답 형태 (Current, Hourly, Daily)
import type { Current, Daily, Hourly } from '../types.js'

export interface NcstItem {
  baseDate: string
  baseTime: string
  category: string
  obsrValue: string
  nx: number
  ny: number
}

export interface FcstItem {
  baseDate: string
  baseTime: string
  category: string
  fcstDate: string
  fcstTime: string
  fcstValue: string
  nx: number
  ny: number
}

export type MidItem = Record<string, string | number>

export interface MidData {
  /** YYYYMMDDHHmm (KST) */
  tmFc: string
  land: MidItem | null
  ta: MidItem | null
}

/** 숫자로 바꾼다. 숫자가 아니거나 결측(-900 이하, 900 이상)이면 null */
export function num(v: unknown): number | null {
  if (v === undefined || v === null || v === '') return null
  const n = typeof v === 'number' ? v : Number(v)
  return Number.isFinite(n) && n > -900 && n < 900 ? n : null
}

function str(v: unknown): string | null {
  return v === undefined || v === null || v === '' ? null : String(v)
}

/** '20261027', '1500' → '2026-10-27T15:00:00+09:00' */
export function kstIso(date: string, time = '0000'): string {
  return `${date.slice(0, 4)}-${date.slice(4, 6)}-${date.slice(6, 8)}T${time.slice(0, 2)}:${time.slice(2, 4)}:00+09:00`
}

function isoDate(date: string): string {
  return `${date.slice(0, 4)}-${date.slice(4, 6)}-${date.slice(6, 8)}`
}

function addDays(date: string, days: number): string {
  const d = new Date(Date.UTC(Number(date.slice(0, 4)), Number(date.slice(4, 6)) - 1, Number(date.slice(6, 8)) + days))
  return `${d.getUTCFullYear()}${String(d.getUTCMonth() + 1).padStart(2, '0')}${String(d.getUTCDate()).padStart(2, '0')}`
}

export function parseCurrent(items: NcstItem[]): Current | null {
  const first = items[0]
  if (!first) return null
  const v = Object.fromEntries(items.map((i) => [i.category, i.obsrValue]))
  return {
    observedAt: kstIso(first.baseDate, first.baseTime),
    temp: num(v.T1H),
    humidity: num(v.REH),
    rain1h: num(v.RN1),
    windSpeed: num(v.WSD),
    windDir: num(v.VEC),
    pty: num(v.PTY),
  }
}

interface Slot {
  date: string
  time: string
  v: Record<string, string>
}

function groupSlots(items: FcstItem[]): Slot[] {
  const slots = new Map<string, Slot>()
  for (const item of items) {
    // TMN/TMX는 06시·15시 칸에 따로 오므로 시간별 목록에는 넣지 않는다
    if (item.category === 'TMN' || item.category === 'TMX') continue
    const key = item.fcstDate + item.fcstTime
    let slot = slots.get(key)
    if (!slot) {
      slot = { date: item.fcstDate, time: item.fcstTime, v: {} }
      slots.set(key, slot)
    }
    slot.v[item.category] = item.fcstValue
  }
  return [...slots.values()].sort((a, b) => (a.date + a.time).localeCompare(b.date + b.time))
}

const PCP_CODE: Record<string, string> = { '1': '약한 비', '2': '보통 비', '3': '강한 비' }
const SNO_CODE: Record<string, string> = { '1': '보통 눈', '2': '많은 눈' }
const WSD_CODE: Record<string, string> = { '1': '약한 바람', '2': '약간 강한 바람', '3': '강한 바람' }

// 연장기간 시각은 PCP·SNO·WSD가 정성 코드다. 날짜로는 구분이 안 돼서(실측: 14시 발표의 +3일은 정량값)
// 정량 구간 PCP가 늘 "강수없음"이나 "…mm"인 점을 이용해 숫자 한 자리면 코드로 본다
function isCoded(v: Record<string, string>): boolean {
  return /^\d$/.test(v.PCP ?? '')
}

function amountLabel(value: string | undefined, coded: boolean, codes: Record<string, string>, none: string): string | null {
  if (!value || value === none || value === '0' || value === '-') return null
  return coded ? (codes[value] ?? null) : value
}

export function parseHourly(items: FcstItem[]): Hourly[] {
  return groupSlots(items).map(({ date, time, v }) => {
    const coded = isCoded(v)
    return {
      time: kstIso(date, time),
      temp: num(v.TMP),
      sky: num(v.SKY),
      pty: num(v.PTY),
      pop: num(v.POP),
      humidity: num(v.REH),
      windDir: num(v.VEC),
      windSpeed: coded ? null : num(v.WSD),
      windText: coded ? (WSD_CODE[v.WSD ?? ''] ?? null) : null,
      pcp: amountLabel(v.PCP, coded, PCP_CODE, '강수없음'),
      sno: amountLabel(v.SNO, coded, SNO_CODE, '적설없음'),
    }
  })
}

const SKY_TEXT: Record<number, string> = { 1: '맑음', 3: '구름많음', 4: '흐림' }
const PTY_TEXT: Record<number, string> = { 1: '비', 2: '비/눈', 3: '눈', 4: '소나기' }

/** 중기예보 문구("구름많고 비" 등)와 같은 형식 */
export function wfText(sky: number | null, pty: number | null): string | null {
  const rain = pty ? PTY_TEXT[pty] : undefined
  if (rain) return `${sky === 4 ? '흐리고' : '구름많고'} ${rain}`
  return sky ? (SKY_TEXT[sky] ?? null) : null
}

function representativeWf(slots: Slot[], preferTime: string): string | null {
  const slot = slots.find((s) => s.time === preferTime) ?? slots[Math.floor(slots.length / 2)]
  return slot ? wfText(num(slot.v.SKY), num(slot.v.PTY)) : null
}

function maxPop(slots: Slot[]): number | null {
  const pops = slots.map((s) => num(s.v.POP)).filter((n): n is number => n !== null)
  return pops.length ? Math.max(...pops) : null
}

export function buildDaily(fcst: FcstItem[], mid: MidData | null): Daily[] {
  const days = new Map<string, Daily>()
  const baseDate = fcst[0]?.baseDate
  const tmn = new Map<string, number | null>()
  const tmx = new Map<string, number | null>()
  for (const item of fcst) {
    if (item.category === 'TMN') tmn.set(item.fcstDate, num(item.fcstValue))
    if (item.category === 'TMX') tmx.set(item.fcstDate, num(item.fcstValue))
  }

  const byDate = new Map<string, Slot[]>()
  for (const slot of groupSlots(fcst)) byDate.set(slot.date, [...(byDate.get(slot.date) ?? []), slot])

  for (const [date, slots] of byDate) {
    // 연장기간 끝의 몇 칸뿐인 날은 중기예보가 더 낫다
    if (date !== baseDate && !tmn.has(date) && !tmx.has(date)) continue
    const temps = slots.map((s) => num(s.v.TMP)).filter((n): n is number => n !== null)
    const am = slots.filter((s) => s.time < '1200')
    const pm = slots.filter((s) => s.time >= '1200')
    days.set(date, {
      date: isoDate(date),
      min: tmn.get(date) ?? (temps.length ? Math.min(...temps) : null),
      max: tmx.get(date) ?? (temps.length ? Math.max(...temps) : null),
      amWf: representativeWf(am, '0900'),
      pmWf: representativeWf(pm, '1500'),
      amPop: maxPop(am),
      pmPop: maxPop(pm),
    })
  }

  if (mid) {
    const start = mid.tmFc.slice(0, 8)
    for (let n = 4; n <= 10; n++) {
      const date = addDays(start, n)
      if (days.has(date)) continue
      // 8일 이후는 오전/오후 구분 없이 하나
      const amKey = n <= 7 ? `${n}Am` : `${n}`
      const pmKey = n <= 7 ? `${n}Pm` : `${n}`
      const entry: Daily = {
        date: isoDate(date),
        min: num(mid.ta?.[`taMin${n}`]),
        max: num(mid.ta?.[`taMax${n}`]),
        amWf: str(mid.land?.[`wf${amKey}`]),
        pmWf: str(mid.land?.[`wf${pmKey}`]),
        amPop: num(mid.land?.[`rnSt${amKey}`]),
        pmPop: num(mid.land?.[`rnSt${pmKey}`]),
      }
      // 18시 발표는 +4일 값이 없다
      if (entry.min === null && entry.max === null && entry.amWf === null) continue
      days.set(date, entry)
    }
  }

  return [...days.values()].sort((a, b) => a.date.localeCompare(b.date))
}
```

- [ ] **Step 4: 테스트 통과 확인**

Run: `pnpm --filter server exec vitest run test/parse.test.ts && pnpm --filter server typecheck`
Expected: 모두 통과

- [ ] **Step 5: 커밋**

```bash
git add server/src/kma/parse.ts server/test/parse.test.ts
git commit -m "feat: 기상청 응답을 현재·시간별·10일 예보로 변환" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: 예보 서비스와 `/weather/api/forecast`

**Files:**
- Create: `server/src/forecast.ts`
- Modify: `server/src/app.ts` (api 라우트 추가)
- Test: `server/test/forecast.test.ts`, `server/test/app.test.ts` (케이스 추가)

**Interfaces:**
- Consumes: Task 2~6 전부. `KmaClient`, `KmaError`, `TtlCache`, `latestIssue`/`previousIssue`/`nextAvailable`/`formatBase`/`formatTmFc`, `toGrid`/`isInGrid`, `findNearestTown`/`townName`/`midLandRegId`/`midTempRegId`, `parseCurrent`/`parseHourly`/`buildDaily`
- Produces: `createForecastService(deps: { kma: KmaClient; now?: () => Date }): ForecastProvider`, `userMessage(err: unknown): string`

- [ ] **Step 1: 실패하는 서비스 테스트 작성**

`server/test/forecast.test.ts`:

```ts
import { readFileSync } from 'node:fs'
import { describe, expect, it } from 'vitest'
import { createForecastService } from '../src/forecast.js'
import { KmaError, parseKmaBody, type KmaClient } from '../src/kma/client.js'

const fixtureItems = (name: string) => parseKmaBody(readFileSync(new URL(`./fixtures/${name}`, import.meta.url), 'utf8'))
const NCST = fixtureItems('ncst.json')
const VILAGE = fixtureItems('vilage.json')
const MID_LAND = fixtureItems('midLand.json')
const MID_TA = fixtureItems('midTa.json')

type Params = Record<string, string | number>
type Handler = (params: Params) => unknown[]

const PATHS = {
  ncst: 'VilageFcstInfoService_2.0/getUltraSrtNcst',
  vilage: 'VilageFcstInfoService_2.0/getVilageFcst',
  land: 'MidFcstInfoService/getMidLandFcst',
  ta: 'MidFcstInfoService/getMidTa',
}

function fakeKma(overrides: Partial<Record<string, Handler>> = {}) {
  const handlers: Record<string, Handler> = {
    [PATHS.ncst]: () => NCST,
    [PATHS.vilage]: () => VILAGE,
    [PATHS.land]: () => MID_LAND,
    [PATHS.ta]: () => MID_TA,
    ...overrides,
  }
  const calls: { path: string; params: Params }[] = []
  const client: KmaClient = {
    async get<T>(path: string, params: Params) {
      calls.push({ path, params })
      const handler = handlers[path]
      if (!handler) throw new KmaError('99', `no handler: ${path}`)
      return handler(params) as T[]
    },
  }
  return { client, calls }
}

const kst = (s: string) => new Date(`${s}+09:00`)
const SEOUL = { lat: 37.5636, lon: 126.98 }

describe('forecast service', () => {
  it('세 API를 불러 응답을 조립한다', async () => {
    const { client, calls } = fakeKma()
    const service = createForecastService({ kma: client, now: () => kst('2026-10-27T15:20:00') })
    const res = await service.get(SEOUL.lat, SEOUL.lon)

    expect(res.location).toMatchObject({ nx: 60, ny: 127 })
    expect(res.location?.name.startsWith('서울특별시')).toBe(true)
    expect(res.current?.temp).toBe(15.3)
    expect(res.hourly).toHaveLength(66)
    expect(res.daily).toHaveLength(11)
    expect(res.stale).toBe(false)
    expect(res.errors).toEqual({})
    expect(res.issued.current).toBe(kst('2026-10-27T15:00:00').toISOString())

    const byPath = (p: string) => calls.find((c) => c.path === p)?.params
    expect(byPath(PATHS.ncst)).toMatchObject({ base_date: '20261027', base_time: '1500', nx: 60, ny: 127 })
    expect(byPath(PATHS.vilage)).toMatchObject({ base_date: '20261027', base_time: '1400', numOfRows: 1000 })
    expect(byPath(PATHS.land)).toMatchObject({ regId: '11B00000', tmFc: '202610270600' })
    expect(byPath(PATHS.ta)).toMatchObject({ regId: '11B10101', tmFc: '202610270600' })
  })

  it('같은 격자는 캐시를 쓴다', async () => {
    const { client, calls } = fakeKma()
    const service = createForecastService({ kma: client, now: () => kst('2026-10-27T15:20:00') })
    await service.get(SEOUL.lat, SEOUL.lon)
    await service.get(SEOUL.lat, SEOUL.lon)
    expect(calls).toHaveLength(4)
  })

  it('최신 발표가 아직 없으면(03) 직전 발표로', async () => {
    const { client } = fakeKma({
      [PATHS.ncst]: (p) => {
        if (p.base_time === '1500') throw new KmaError('03', 'NO_DATA')
        return NCST
      },
    })
    const service = createForecastService({ kma: client, now: () => kst('2026-10-27T15:20:00') })
    const res = await service.get(SEOUL.lat, SEOUL.lon)
    expect(res.issued.current).toBe(kst('2026-10-27T14:00:00').toISOString())
    expect(res.errors.current).toBeUndefined()
  })

  it('일부 실패는 해당 섹션만 오류', async () => {
    const { client } = fakeKma({
      [PATHS.vilage]: () => {
        throw new KmaError('22', 'LIMITED')
      },
    })
    const service = createForecastService({ kma: client, now: () => kst('2026-10-27T15:20:00') })
    const res = await service.get(SEOUL.lat, SEOUL.lon)
    expect(res.errors.hourly).toContain('한도')
    expect(res.hourly).toEqual([])
    expect(res.current?.temp).toBe(15.3)
    expect(res.daily).toHaveLength(7)
  })

  it('만료 후 실패하면 이전 값을 stale로', async () => {
    let now = kst('2026-10-27T15:20:00')
    let down = false
    const { client } = fakeKma({
      [PATHS.ncst]: () => {
        if (down) throw new KmaError('NETWORK', 'down')
        return NCST
      },
    })
    const service = createForecastService({ kma: client, now: () => now })
    await service.get(SEOUL.lat, SEOUL.lon)
    now = kst('2026-10-27T16:15:00') // 초단기실황 16시 발표가 16:10부터 조회 가능 → 만료
    down = true
    const res = await service.get(SEOUL.lat, SEOUL.lon)
    expect(res.stale).toBe(true)
    expect(res.current?.temp).toBe(15.3)
  })

  it('남한 밖은 기상청을 부르지 않는다', async () => {
    const { client, calls } = fakeKma()
    const service = createForecastService({ kma: client })
    const res = await service.get(35, 120)
    expect(res.location).toBeNull()
    expect(calls).toHaveLength(0)
  })
})
```

- [ ] **Step 2: 테스트 실패 확인**

Run: `pnpm --filter server exec vitest run test/forecast.test.ts`
Expected: FAIL (모듈 없음)

- [ ] **Step 3: 서비스 구현**

`server/src/forecast.ts`:

```ts
// 위치 → 격자·중기구역 → 기상청 3종 병렬 호출(캐시) → ForecastResponse
import type { ForecastProvider } from './app.js'
import { TtlCache, type CacheResult, type Loaded } from './cache.js'
import { formatBase, formatTmFc, latestIssue, nextAvailable, previousIssue, type IssueKind } from './kma/baseTime.js'
import { KmaError, type KmaClient } from './kma/client.js'
import { isInGrid, toGrid } from './kma/grid.js'
import { buildDaily, parseCurrent, parseHourly, type FcstItem, type MidData, type MidItem, type NcstItem } from './kma/parse.js'
import { findNearestTown, midLandRegId, midTempRegId, townName } from './regions.js'
import type { ForecastResponse } from './types.js'

const MAX_TOWN_DISTANCE_KM = 20
const NODATA_RETRY_MS = 5 * 60_000

interface Issued<T> {
  issue: Date
  data: T
}

type MidPair = { land: MidItem | null; ta: MidItem | null }

export function userMessage(err: unknown): string {
  if (err instanceof KmaError) {
    if (err.code === '22') return '기상청 호출 한도를 넘었습니다. 잠시 후 다시 시도해 주세요.'
    if (err.code === 'NETWORK') return '기상청 서버에 연결하지 못했습니다.'
    if (err.code === '03') return '아직 발표된 예보가 없습니다.'
  }
  return '기상청 데이터를 가져오지 못했습니다.'
}

function baseParams(issue: Date) {
  const { baseDate, baseTime } = formatBase(issue)
  return { base_date: baseDate, base_time: baseTime }
}

function emptyResponse(): ForecastResponse {
  return { location: null, current: null, hourly: [], daily: [], issued: {}, stale: false, errors: {} }
}

export function createForecastService({ kma, now = () => new Date() }: { kma: KmaClient; now?: () => Date }): ForecastProvider {
  const ncstCache = new TtlCache<Issued<NcstItem[]>>(5000, () => now().getTime())
  const vilageCache = new TtlCache<Issued<FcstItem[]>>(5000, () => now().getTime())
  const midCache = new TtlCache<Issued<MidPair>>(500, () => now().getTime())

  // 최신 발표가 아직 없으면(03 NODATA) 직전 발표로 한 번 더 시도한다
  async function loadIssued<T>(kind: IssueKind, fetchAt: (issue: Date) => Promise<T>): Promise<Loaded<Issued<T>>> {
    const current = now()
    const latest = latestIssue(kind, current)
    try {
      return { value: { issue: latest, data: await fetchAt(latest) }, expiresAt: nextAvailable(kind, latest).getTime() }
    } catch (err) {
      if (!(err instanceof KmaError && err.code === '03')) throw err
      const prev = previousIssue(kind, latest)
      return {
        value: { issue: prev, data: await fetchAt(prev) },
        expiresAt: Math.max(nextAvailable(kind, prev).getTime(), current.getTime() + NODATA_RETRY_MS),
      }
    }
  }

  function settle<T>(section: string, result: PromiseSettledResult<CacheResult<Issued<T>>>): CacheResult<Issued<T>> | Error {
    if (result.status === 'fulfilled') return result.value
    const err = result.reason as Error
    console.warn(`[kma] ${section} 실패`, err instanceof KmaError ? err.code : '', err.message)
    return err
  }

  return {
    async get(lat, lon) {
      const grid = toGrid(lat, lon)
      const nearest = findNearestTown(lat, lon)
      if (!isInGrid(grid) || nearest.distanceKm > MAX_TOWN_DISTANCE_KM) return emptyResponse()

      const { nx, ny } = grid
      const land = midLandRegId(nearest.town)
      const ta = land ? midTempRegId(nearest.town, land) : null
      const gridKey = `${nx},${ny}`

      const [ncstResult, vilageResult, midResult] = await Promise.allSettled([
        ncstCache.getOrLoad(gridKey, () =>
          loadIssued('ncst', (issue) =>
            kma.get<NcstItem>('VilageFcstInfoService_2.0/getUltraSrtNcst', { numOfRows: 20, ...baseParams(issue), nx, ny }),
          ),
        ),
        vilageCache.getOrLoad(gridKey, () =>
          loadIssued('vilage', (issue) =>
            kma.get<FcstItem>('VilageFcstInfoService_2.0/getVilageFcst', { numOfRows: 1000, ...baseParams(issue), nx, ny }),
          ),
        ),
        land && ta
          ? midCache.getOrLoad(`${land},${ta}`, () =>
              loadIssued('mid', async (issue) => {
                const tmFc = formatTmFc(issue)
                const [landItems, taItems] = await Promise.all([
                  kma.get<MidItem>('MidFcstInfoService/getMidLandFcst', { numOfRows: 10, regId: land, tmFc }),
                  kma.get<MidItem>('MidFcstInfoService/getMidTa', { numOfRows: 10, regId: ta, tmFc }),
                ])
                return { land: landItems[0] ?? null, ta: taItems[0] ?? null }
              }),
            )
          : Promise.reject(new KmaError('REGION', '중기예보 구역을 찾지 못했습니다')),
      ])

      const res: ForecastResponse = {
        location: { lat, lon, nx, ny, name: townName(nearest.town) },
        current: null,
        hourly: [],
        daily: [],
        issued: {},
        stale: false,
        errors: {},
      }

      const ncst = settle('초단기실황', ncstResult)
      if (ncst instanceof Error) res.errors.current = userMessage(ncst)
      else {
        res.current = parseCurrent(ncst.value.data)
        res.issued.current = ncst.value.issue.toISOString()
        res.stale ||= ncst.stale
      }

      const vilage = settle('단기예보', vilageResult)
      let fcstItems: FcstItem[] = []
      if (vilage instanceof Error) res.errors.hourly = userMessage(vilage)
      else {
        fcstItems = vilage.value.data
        res.hourly = parseHourly(fcstItems)
        res.issued.hourly = vilage.value.issue.toISOString()
        res.stale ||= vilage.stale
      }

      const mid = settle('중기예보', midResult)
      let midData: MidData | null = null
      if (mid instanceof Error) res.errors.daily = userMessage(mid)
      else {
        midData = { tmFc: formatTmFc(mid.value.issue), ...mid.value.data }
        res.issued.mid = mid.value.issue.toISOString()
        res.stale ||= mid.stale
      }

      res.daily = buildDaily(fcstItems, midData)
      return res
    },
  }
}
```

- [ ] **Step 4: 서비스 테스트 통과 확인**

Run: `pnpm --filter server exec vitest run test/forecast.test.ts`
Expected: 6 passed

- [ ] **Step 5: 라우트 테스트 추가 (실패 확인용)**

`server/test/app.test.ts` 전체를 아래로 바꾼다:

```ts
import { describe, expect, it } from 'vitest'
import { createApp, type ForecastProvider } from '../src/app.js'
import type { ForecastResponse } from '../src/types.js'

const EMPTY: ForecastResponse = { location: null, current: null, hourly: [], daily: [], issued: {}, stale: false, errors: {} }

function provider(): ForecastProvider & { calls: [number, number][] } {
  const calls: [number, number][] = []
  return {
    calls,
    async get(lat, lon) {
      calls.push([lat, lon])
      return EMPTY
    },
  }
}

describe('app', () => {
  it('GET /weather/healthz 는 ok를 돌려준다', async () => {
    const res = await createApp({ forecast: null }).request('/weather/healthz')
    expect(res.status).toBe(200)
    expect(await res.json()).toEqual({ ok: true })
  })

  it('키가 없으면 api는 503', async () => {
    const res = await createApp({ forecast: null }).request('/weather/api/forecast?lat=37.5&lon=127')
    expect(res.status).toBe(503)
  })

  it('lat/lon이 없거나 잘못되면 400', async () => {
    const app = createApp({ forecast: provider() })
    expect((await app.request('/weather/api/forecast?lat=37.5')).status).toBe(400)
    expect((await app.request('/weather/api/forecast?lat=abc&lon=127')).status).toBe(400)
    expect((await app.request('/weather/api/forecast?lat=95&lon=127')).status).toBe(400)
  })

  it('정상 요청은 서비스 결과를 그대로', async () => {
    const p = provider()
    const res = await createApp({ forecast: p }).request('/weather/api/forecast?lat=37.5&lon=127.01')
    expect(res.status).toBe(200)
    expect(res.headers.get('cache-control')).toBe('no-store')
    expect(await res.json()).toEqual(EMPTY)
    expect(p.calls).toEqual([[37.5, 127.01]])
  })
})
```

Run: `pnpm --filter server exec vitest run test/app.test.ts`
Expected: FAIL (503·400 케이스가 404)

- [ ] **Step 6: 라우트 구현**

`server/src/app.ts`의 `app.get('/healthz', ...)` 줄 바로 아래에 추가:

```ts
  app.get('/api/forecast', async (c) => {
    if (!deps.forecast) return c.json({ error: '서버에 KMA_SERVICE_KEY가 설정되지 않았습니다.' }, 503)
    const latRaw = c.req.query('lat')
    const lonRaw = c.req.query('lon')
    const lat = Number(latRaw)
    const lon = Number(lonRaw)
    if (!latRaw || !lonRaw || !Number.isFinite(lat) || !Number.isFinite(lon) || Math.abs(lat) > 90 || Math.abs(lon) > 180) {
      return c.json({ error: 'lat, lon 값이 올바르지 않습니다.' }, 400)
    }
    c.header('Cache-Control', 'no-store')
    return c.json(await deps.forecast.get(lat, lon))
  })
```

- [ ] **Step 7: 전체 서버 테스트 통과 확인**

Run: `pnpm --filter server test && pnpm --filter server typecheck`
Expected: 모두 통과

- [ ] **Step 8: 커밋**

```bash
git add server/src/forecast.ts server/src/app.ts server/test/forecast.test.ts server/test/app.test.ts
git commit -m "feat: 예보 서비스와 /weather/api/forecast" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: 정적 파일 서빙과 실행 진입점

**Files:**
- Create: `server/src/static.ts`
- Modify: `server/src/app.ts` (정적 라우트), `server/src/index.ts` (의존성 조립)
- Test: `server/test/static.test.ts`

**Interfaces:**
- Consumes: `createForecastService`, `createKmaClient`
- Produces: `readStatic(root: string, relPath: string): Promise<StaticFile | null>`, `interface StaticFile { body: ArrayBuffer; type: string }`. 라우트: `GET /weather/data/wind/*`(latest.json은 `no-cache`, 나머지는 `public, max-age=86400`), `GET /weather/*`(파일, `assets/`는 immutable; 확장자 없는 경로는 index.html로 SPA fallback, 확장자 있는데 없으면 404), `/weather/api/*` 나머지는 JSON 404.

- [ ] **Step 1: 실패하는 테스트 작성**

`server/test/static.test.ts`:

```ts
import { mkdirSync, mkdtempSync, writeFileSync } from 'node:fs'
import { tmpdir } from 'node:os'
import { join } from 'node:path'
import { beforeAll, describe, expect, it } from 'vitest'
import { createApp } from '../src/app.js'

let app: ReturnType<typeof createApp>

beforeAll(() => {
  const root = mkdtempSync(join(tmpdir(), 'weather-static-'))
  const web = join(root, 'web')
  const wind = join(root, 'wind')
  mkdirSync(join(web, 'assets'), { recursive: true })
  mkdirSync(join(wind, '2026102706'), { recursive: true })
  writeFileSync(join(web, 'index.html'), '<!doctype html><title>t</title>')
  writeFileSync(join(web, 'assets', 'app.js'), 'console.log(1)')
  writeFileSync(join(wind, 'latest.json'), '{"cycle":"2026102706"}')
  writeFileSync(join(wind, '2026102706', 'f000.png'), 'png')
  writeFileSync(join(root, 'secret.txt'), 'nope')
  app = createApp({ forecast: null, webDir: web, windDir: wind })
})

describe('정적 파일', () => {
  it('/weather/ 는 index.html (no-cache)', async () => {
    const res = await app.request('/weather/')
    expect(res.status).toBe(200)
    expect(res.headers.get('content-type')).toContain('text/html')
    expect(res.headers.get('cache-control')).toBe('no-cache')
  })

  it('assets는 오래 캐시', async () => {
    const res = await app.request('/weather/assets/app.js')
    expect(res.status).toBe(200)
    expect(res.headers.get('content-type')).toContain('text/javascript')
    expect(res.headers.get('cache-control')).toContain('immutable')
  })

  it('확장자 없는 경로는 SPA fallback, 있는데 없으면 404', async () => {
    expect((await app.request('/weather/some/route')).headers.get('content-type')).toContain('text/html')
    expect((await app.request('/weather/missing.js')).status).toBe(404)
  })

  it('바람 데이터', async () => {
    const latest = await app.request('/weather/data/wind/latest.json')
    expect(latest.status).toBe(200)
    expect(latest.headers.get('cache-control')).toBe('no-cache')
    expect(await latest.json()).toEqual({ cycle: '2026102706' })
    const png = await app.request('/weather/data/wind/2026102706/f000.png')
    expect(png.headers.get('content-type')).toBe('image/png')
    expect(png.headers.get('cache-control')).toBe('public, max-age=86400')
    expect((await app.request('/weather/data/wind/nope.png')).status).toBe(404)
  })

  it('root 밖은 읽지 않는다', async () => {
    expect((await app.request('/weather/data/wind/%2e%2e%2fsecret.txt')).status).toBe(404)
    expect((await app.request('/weather/%2e%2e%2fsecret.txt')).status).toBe(404)
  })

  it('없는 api는 JSON 404', async () => {
    const res = await app.request('/weather/api/nope')
    expect(res.status).toBe(404)
    expect(res.headers.get('content-type')).toContain('application/json')
  })
})
```

- [ ] **Step 2: 테스트 실패 확인**

Run: `pnpm --filter server exec vitest run test/static.test.ts`
Expected: FAIL (404 등)

- [ ] **Step 3: readStatic 구현**

`server/src/static.ts`:

```ts
// root 아래 파일만 읽는 정적 파일 헬퍼 (root 밖을 가리키면 null)
import { readFile, stat } from 'node:fs/promises'
import { extname, join, resolve, sep } from 'node:path'

export const MIME: Record<string, string> = {
  '.html': 'text/html; charset=utf-8',
  '.js': 'text/javascript; charset=utf-8',
  '.css': 'text/css; charset=utf-8',
  '.json': 'application/json; charset=utf-8',
  '.png': 'image/png',
  '.svg': 'image/svg+xml',
  '.ico': 'image/x-icon',
  '.webmanifest': 'application/manifest+json',
  '.woff2': 'font/woff2',
  '.txt': 'text/plain; charset=utf-8',
}

export interface StaticFile {
  body: ArrayBuffer
  type: string
}

export async function readStatic(root: string, relPath: string): Promise<StaticFile | null> {
  const base = resolve(root)
  let decoded: string
  try {
    decoded = decodeURIComponent(relPath)
  } catch {
    return null
  }
  const full = resolve(join(base, decoded))
  if (full !== base && !full.startsWith(base + sep)) return null
  try {
    const info = await stat(full)
    if (!info.isFile()) return null
    const buf = await readFile(full)
    return {
      body: buf.buffer.slice(buf.byteOffset, buf.byteOffset + buf.byteLength) as ArrayBuffer,
      type: MIME[extname(full).toLowerCase()] ?? 'application/octet-stream',
    }
  } catch {
    return null
  }
}
```

- [ ] **Step 4: app에 정적 라우트 추가**

`server/src/app.ts` 맨 위 import에 추가:

```ts
import { extname } from 'node:path'
import { readStatic, type StaticFile } from './static.js'
```

`createApp` 위에 헬퍼 추가:

```ts
function fileResponse(file: StaticFile, cacheControl: string): Response {
  return new Response(file.body, { headers: { 'Content-Type': file.type, 'Cache-Control': cacheControl } })
}
```

`/api/forecast` 라우트 아래, `return app` 위에 추가:

```ts
  app.all('/api/*', (c) => c.json({ error: 'not found' }, 404))

  app.get('/data/wind/*', async (c) => {
    const rel = c.req.path.slice('/weather/data/wind/'.length)
    const file = deps.windDir ? await readStatic(deps.windDir, rel) : null
    if (!file) return c.json({ error: 'not found' }, 404)
    // latest.json만 자주 바뀌고, 프레임 PNG는 사이클 폴더 경로라 내용이 바뀌지 않는다
    return fileResponse(file, rel === 'latest.json' ? 'no-cache' : 'public, max-age=86400')
  })

  app.get('*', async (c) => {
    if (!deps.webDir) return c.notFound()
    const rel = c.req.path.replace(/^\/weather\/?/, '') || 'index.html'
    const file = await readStatic(deps.webDir, rel)
    if (file) return fileResponse(file, rel.startsWith('assets/') ? 'public, max-age=31536000, immutable' : 'no-cache')
    // 확장자가 있는 경로(빠진 js 등)는 index.html로 대신하지 않는다
    if (extname(rel)) return c.notFound()
    const index = await readStatic(deps.webDir, 'index.html')
    return index ? fileResponse(index, 'no-cache') : c.notFound()
  })
```

- [ ] **Step 5: 테스트 통과 확인**

Run: `pnpm --filter server test && pnpm --filter server typecheck`
Expected: 모두 통과

- [ ] **Step 6: 진입점에서 의존성 조립**

`server/src/index.ts` 전체를 아래로 바꾼다:

```ts
import { serve } from '@hono/node-server'
import { join, resolve } from 'node:path'
import { fileURLToPath } from 'node:url'
import { createApp } from './app.js'
import { createForecastService } from './forecast.js'
import { createKmaClient } from './kma/client.js'

// src/index.ts, dist/index.js 어느 쪽에서 실행해도 저장소 루트를 가리킨다
const ROOT = fileURLToPath(new URL('../../', import.meta.url))
// 이미 있는 환경변수(pm2 env)는 덮어쓰지 않는다
try {
  process.loadEnvFile(join(ROOT, '.env'))
} catch {
  // .env가 없으면 환경변수만 쓴다
}

const serviceKey = process.env.KMA_SERVICE_KEY?.trim()
if (!serviceKey) console.warn('[weather] KMA_SERVICE_KEY가 없어 /weather/api/forecast는 503을 반환합니다')

const app = createApp({
  forecast: serviceKey ? createForecastService({ kma: createKmaClient({ serviceKey }) }) : null,
  webDir: join(ROOT, 'web/dist'),
  windDir: resolve(ROOT, process.env.WIND_DATA_DIR ?? 'data/wind'),
})

const port = Number(process.env.PORT ?? 3090)
serve({ fetch: app.fetch, port, hostname: '127.0.0.1' }, () => {
  console.log(`[weather] http://127.0.0.1:${port}/weather/`)
})
```

- [ ] **Step 7: 실제 기상청으로 확인**

Run (터미널 1): `pnpm --filter server dev`
Expected: `[weather] http://127.0.0.1:3090/weather/`

Run (터미널 2): `curl -s 'http://127.0.0.1:3090/weather/api/forecast?lat=37.5636&lon=126.98' | head -c 600`
Expected: `"location":{"lat":37.5636,"lon":126.98,"nx":60,"ny":127,"name":"서울특별시..."`, `current`에 실제 기온, `errors`가 `{}`. 응답 어디에도 서비스 키가 없다. 확인 후 서버를 끈다.

- [ ] **Step 8: 빌드 확인과 커밋**

Run: `pnpm --filter server build && ls server/dist/data`
Expected: `midTempRegions.json  towns.json` (tsc가 import한 JSON을 복사함)

```bash
git add server/src/static.ts server/src/app.ts server/src/index.ts server/test/static.test.ts
git commit -m "feat: 웹·바람 정적 파일 서빙과 서버 진입점" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: GFS 바람 배치 (Python)

**Files:**
- Create: `wind/requirements.txt`, `wind/requirements-dev.txt`, `wind/pytest.ini`, `wind/wind_job.py`
- Test: `wind/tests/test_wind_job.py` (fixture `wind/tests/fixtures/gfs-10m-f003.grb2`는 이미 있음)

**Interfaces:**
- Produces: `data/wind/latest.json`과 `data/wind/<cycle>/fNNN.png`. 형식:
  ```json
  { "cycle": "2026102706", "bbox": [110.0, 20.0, 150.0, 50.0], "width": 161, "height": 121,
    "frames": [{ "time": "2026-10-27T06:00:00Z", "file": "2026102706/f000.png", "uMin": -14.3, "uMax": 14.1, "vMin": -15.2, "vMax": 12.2 }] }
  ```
  PNG: RGBA, 행 0이 북쪽(50°N), 열 0이 서쪽(110°E). R = (u−uMin)/(uMax−uMin)·255, G는 v에 대해 같은 방식, B=0, A=255.
- 환경변수 `WIND_DATA_DIR` (기본 `data/wind`, 실행 위치 기준)

- [ ] **Step 1: 의존성 파일과 venv**

`wind/requirements.txt`:

```
eccodes==2.50.0
numpy==2.5.4
pillow==13.0.0
```

`wind/requirements-dev.txt`:

```
-r requirements.txt
pytest>=9
```

`wind/pytest.ini`:

```ini
[pytest]
pythonpath = .
testpaths = tests
```

Run: `python3 -m venv wind/.venv && wind/.venv/bin/pip install -q -r wind/requirements-dev.txt`
Expected: 오류 없이 설치 (macOS arm64, Python 3.14에서 확인됨)

- [ ] **Step 2: 실패하는 테스트 작성**

`wind/tests/test_wind_job.py`:

```python
import io
import json
import tempfile
from datetime import datetime, timezone
from pathlib import Path
from urllib.parse import unquote

import eccodes
import numpy as np
import pytest
from PIL import Image

import wind_job

FIXTURE = (Path(__file__).parent / "fixtures" / "gfs-10m-f003.grb2").read_bytes()
NOW = datetime(2026, 10, 27, 7, 40, tzinfo=timezone.utc)


def test_candidate_cycles_newest_first():
    cycles = wind_job.candidate_cycles(NOW)
    assert [wind_job.cycle_id(c) for c in cycles] == ["2026102706", "2026102700", "2026102618", "2026102612"]


def test_grib_url():
    url = unquote(wind_job.grib_url(datetime(2026, 10, 27, 6, tzinfo=timezone.utc), 3))
    assert url.startswith("https://nomads.ncep.noaa.gov/cgi-bin/filter_gfs_0p25.pl?")
    assert "dir=/gfs.20261027/06/atmos" in url
    assert "file=gfs.t06z.pgrb2.0p25.f003" in url
    assert "var_UGRD=on" in url and "var_VGRD=on" in url and "lev_10_m_above_ground=on" in url
    assert "leftlon=110" in url and "rightlon=150" in url and "toplat=50" in url and "bottomlat=20" in url


def raw_first_message_values(grib: bytes) -> np.ndarray:
    with tempfile.TemporaryFile() as f:
        f.write(grib)
        f.seek(0)
        h = eccodes.codes_grib_new_from_file(f)
        try:
            return eccodes.codes_get_values(h).reshape(eccodes.codes_get(h, "Nj"), eccodes.codes_get(h, "Ni"))
        finally:
            eccodes.codes_release(h)


def test_decode_uv_is_north_up():
    u, v = wind_job.decode_uv(FIXTURE)
    assert u.shape == (121, 161) and v.shape == (121, 161)
    raw = raw_first_message_values(FIXTURE)  # 원본은 남→북 순서
    np.testing.assert_allclose(u[0], raw[-1], rtol=1e-5)
    np.testing.assert_allclose(u[-1], raw[0], rtol=1e-5)


def test_encode_png_roundtrip():
    u = np.linspace(-10, 20, 6, dtype=np.float32).reshape(2, 3)
    v = np.linspace(5, -5, 6, dtype=np.float32).reshape(2, 3)
    png, ranges = wind_job.encode_png(u, v)
    assert ranges == {"uMin": -10.0, "uMax": 20.0, "vMin": -5.0, "vMax": 5.0}
    img = np.asarray(Image.open(io.BytesIO(png)).convert("RGBA")).astype(np.float32)
    assert img.shape == (2, 3, 4)
    u_back = ranges["uMin"] + img[..., 0] / 255 * (ranges["uMax"] - ranges["uMin"])
    v_back = ranges["vMin"] + img[..., 1] / 255 * (ranges["vMax"] - ranges["vMin"])
    np.testing.assert_allclose(u_back, u, atol=30 / 255)
    np.testing.assert_allclose(v_back, v, atol=10 / 255)
    assert (img[..., 3] == 255).all()


def test_run_writes_cycle_and_skips_when_current(tmp_path: Path):
    msg = wind_job.run(tmp_path, NOW, fetch=lambda url: FIXTURE, pause=0)
    assert "2026102706" in msg
    latest = json.loads((tmp_path / "latest.json").read_text())
    assert latest["cycle"] == "2026102706"
    assert latest["bbox"] == [110.0, 20.0, 150.0, 50.0]
    assert (latest["width"], latest["height"]) == (161, 121)
    assert len(latest["frames"]) == 25
    assert latest["frames"][0]["time"] == "2026-10-27T06:00:00Z"
    assert latest["frames"][-1]["file"] == "2026102706/f072.png"
    assert (tmp_path / "2026102706" / "f072.png").exists()
    assert "유지" in wind_job.run(tmp_path, NOW, fetch=lambda url: pytest.fail("다시 받으면 안 됨"), pause=0)


def test_run_falls_back_to_previous_cycle(tmp_path: Path):
    def fetch(url: str):
        return None if "gfs.20261027/06" in unquote(url) else FIXTURE

    wind_job.run(tmp_path, NOW, fetch=fetch, pause=0)
    assert json.loads((tmp_path / "latest.json").read_text())["cycle"] == "2026102700"


def test_prune_keeps_two_newest(tmp_path: Path):
    for cid in ["2026102612", "2026102618", "2026102700"]:
        (tmp_path / cid).mkdir()
    wind_job.prune(tmp_path, keep=2)
    assert sorted(p.name for p in tmp_path.iterdir()) == ["2026102618", "2026102700"]
```

- [ ] **Step 3: 테스트 실패 확인**

Run: `cd wind && .venv/bin/python -m pytest -q; cd ..`
Expected: FAIL (`ModuleNotFoundError: No module named 'wind_job'`)

- [ ] **Step 4: 구현**

`wind/wind_job.py`:

```python
"""GFS 10m 바람(동아시아)을 받아 프레임별 PNG와 latest.json을 만든다.

pm2 앱 weather-wind가 매시 15분에 실행한다. 새 사이클이 없으면 바로 끝난다.
PNG는 R=U, G=V를 프레임별 min~max로 0~255에 담고, 행 0이 북쪽이다.
"""

from __future__ import annotations

import io
import json
import os
import shutil
import tempfile
import time
import urllib.error
import urllib.parse
import urllib.request
from datetime import datetime, timedelta, timezone
from pathlib import Path
from typing import Callable

import eccodes
import numpy as np
from PIL import Image

BBOX = (110.0, 20.0, 150.0, 50.0)  # west, south, east, north
FORECAST_HOURS = list(range(0, 73, 3))
KEEP_CYCLES = 2
NOMADS = "https://nomads.ncep.noaa.gov/cgi-bin/filter_gfs_0p25.pl"
USER_AGENT = "bytebard-weather/1.0 (+https://bytebard.cloud/weather)"

Fetch = Callable[[str], "bytes | None"]


def candidate_cycles(now: datetime, count: int = 4) -> list[datetime]:
    """now 이전의 GFS 사이클(00/06/12/18Z)을 최신순으로"""
    base = now.astimezone(timezone.utc).replace(minute=0, second=0, microsecond=0)
    base = base.replace(hour=base.hour - base.hour % 6)
    return [base - timedelta(hours=6 * i) for i in range(count)]


def cycle_id(cycle: datetime) -> str:
    return cycle.strftime("%Y%m%d%H")


def grib_url(cycle: datetime, fhour: int) -> str:
    day, hour = cycle.strftime("%Y%m%d"), cycle.strftime("%H")
    west, south, east, north = BBOX
    params = {
        "dir": f"/gfs.{day}/{hour}/atmos",
        "file": f"gfs.t{hour}z.pgrb2.0p25.f{fhour:03d}",
        "var_UGRD": "on",
        "var_VGRD": "on",
        "lev_10_m_above_ground": "on",
        "subregion": "",
        "leftlon": f"{west:g}",
        "rightlon": f"{east:g}",
        "toplat": f"{north:g}",
        "bottomlat": f"{south:g}",
    }
    return f"{NOMADS}?{urllib.parse.urlencode(params)}"


def download(url: str, retries: int = 2) -> bytes | None:
    """GRIB을 받는다. 아직 공개되지 않은 파일이면 None"""
    for attempt in range(retries + 1):
        try:
            req = urllib.request.Request(url, headers={"User-Agent": USER_AGENT})
            with urllib.request.urlopen(req, timeout=60) as res:
                data = res.read()
            # 없는 파일에 200 + HTML 오류 페이지를 줄 때가 있다
            return data if data[:4] == b"GRIB" else None
        except urllib.error.HTTPError as err:
            if err.code == 404:
                return None
            if attempt == retries:
                raise
        except urllib.error.URLError:
            if attempt == retries:
                raise
        time.sleep(5 * (attempt + 1))
    return None


def decode_uv(grib: bytes) -> tuple[np.ndarray, np.ndarray]:
    """10m U/V 배열 [행=북→남, 열=서→동]"""
    fields: dict[str, np.ndarray] = {}
    # eccodes는 실제 파일 핸들이 필요하다
    with tempfile.TemporaryFile() as f:
        f.write(grib)
        f.seek(0)
        while (h := eccodes.codes_grib_new_from_file(f)) is not None:
            try:
                ni, nj = eccodes.codes_get(h, "Ni"), eccodes.codes_get(h, "Nj")
                values = eccodes.codes_get_values(h).reshape(nj, ni)
                if eccodes.codes_get(h, "jScansPositively") == 1:
                    values = values[::-1]
                fields[eccodes.codes_get(h, "shortName")] = values.astype(np.float32)
            finally:
                eccodes.codes_release(h)
    return fields["10u"], fields["10v"]


def encode_png(u: np.ndarray, v: np.ndarray) -> tuple[bytes, dict[str, float]]:
    def scale(a: np.ndarray) -> tuple[np.ndarray, float, float]:
        lo, hi = float(a.min()), float(a.max())
        span = (hi - lo) or 1.0
        return np.round((a - lo) / span * 255).astype(np.uint8), lo, hi

    r, umin, umax = scale(u)
    g, vmin, vmax = scale(v)
    rgba = np.dstack([r, g, np.zeros_like(r), np.full_like(r, 255)])
    buf = io.BytesIO()
    Image.fromarray(rgba, "RGBA").save(buf, format="PNG")
    ranges = {"uMin": round(umin, 3), "uMax": round(umax, 3), "vMin": round(vmin, 3), "vMax": round(vmax, 3)}
    return buf.getvalue(), ranges


def fetch_cycle(cycle: datetime, fetch: Fetch, pause: float):
    """사이클의 모든 프레임을 받아 PNG로. 하나라도 없으면 None"""
    # 마지막 프레임이 있으면 사이클 전체가 공개된 것으로 본다
    if fetch(grib_url(cycle, FORECAST_HOURS[-1])) is None:
        return None
    frames = []
    size = (0, 0)
    for fhour in FORECAST_HOURS:
        grib = fetch(grib_url(cycle, fhour))
        if grib is None:
            return None
        u, v = decode_uv(grib)
        png, ranges = encode_png(u, v)
        frames.append((fhour, png, ranges))
        size = (u.shape[1], u.shape[0])
        time.sleep(pause)  # NOMADS 요청 제한을 지킨다
    return frames, size


def prune(out_dir: Path, keep: int = KEEP_CYCLES) -> None:
    cycles = sorted((p for p in out_dir.iterdir() if p.is_dir() and p.name.isdigit()), key=lambda p: p.name, reverse=True)
    for old in cycles[keep:]:
        shutil.rmtree(old)


def write_cycle(out_dir: Path, cycle: datetime, frames, width: int, height: int) -> None:
    cid = cycle_id(cycle)
    tmp_dir = out_dir / f".{cid}.tmp"
    shutil.rmtree(tmp_dir, ignore_errors=True)
    tmp_dir.mkdir(parents=True)
    meta_frames = []
    for fhour, png, ranges in frames:
        name = f"f{fhour:03d}.png"
        (tmp_dir / name).write_bytes(png)
        valid = cycle + timedelta(hours=fhour)
        meta_frames.append({"time": valid.strftime("%Y-%m-%dT%H:%M:%SZ"), "file": f"{cid}/{name}", **ranges})
    final_dir = out_dir / cid
    shutil.rmtree(final_dir, ignore_errors=True)
    tmp_dir.rename(final_dir)
    latest = {"cycle": cid, "bbox": list(BBOX), "width": width, "height": height, "frames": meta_frames}
    # 프레임이 다 준비된 뒤에 latest.json을 원자적으로 바꾼다
    tmp_json = out_dir / "latest.json.tmp"
    tmp_json.write_text(json.dumps(latest))
    os.replace(tmp_json, out_dir / "latest.json")
    prune(out_dir)


def current_cycle(out_dir: Path) -> str | None:
    try:
        return json.loads((out_dir / "latest.json").read_text())["cycle"]
    except (FileNotFoundError, KeyError, json.JSONDecodeError):
        return None


def run(out_dir: Path, now: datetime, fetch: Fetch = download, pause: float = 1.0) -> str:
    out_dir.mkdir(parents=True, exist_ok=True)
    have = current_cycle(out_dir)
    for cycle in candidate_cycles(now):
        cid = cycle_id(cycle)
        if cid == have:
            return f"최신 사이클 {cid} 유지"
        result = fetch_cycle(cycle, fetch, pause)
        if result is None:
            continue
        frames, (width, height) = result
        write_cycle(out_dir, cycle, frames, width, height)
        return f"사이클 {cid} 저장 ({len(frames)}프레임)"
    return "받을 수 있는 사이클이 없음"


def main() -> None:
    out_dir = Path(os.environ.get("WIND_DATA_DIR", "data/wind")).resolve()
    print(run(out_dir, datetime.now(timezone.utc)), flush=True)


if __name__ == "__main__":
    main()
```

- [ ] **Step 5: 테스트 통과 확인**

Run: `cd wind && .venv/bin/python -m pytest -q; cd ..`
Expected: 7 passed

- [ ] **Step 6: 실제 NOMADS로 한 번 실행 (로컬 개발 데이터)**

Run: `WIND_DATA_DIR=data/wind wind/.venv/bin/python wind/wind_job.py`
Expected: 약 1~2분 뒤 `사이클 YYYYMMDDHH 저장 (25프레임)`. `data/wind/latest.json`이 생긴다(`data/wind/`는 gitignore 대상).

- [ ] **Step 7: 커밋**

```bash
git add wind/requirements.txt wind/requirements-dev.txt wind/pytest.ini wind/wind_job.py wind/tests
git commit -m "feat: GFS 10m 바람 PNG 배치" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: 웹 뼈대와 지도

**Files:**
- Create: `web/package.json`, `web/tsconfig.json`, `web/vite.config.ts`, `web/index.html`
- Create: `web/src/main.tsx`, `web/src/App.tsx`, `web/src/styles.css`, `web/src/map/MapView.tsx`
- Create: `web/src/format.ts`
- Test: `web/src/format.test.ts`

**Interfaces:**
- Produces:
  - `MapView(props: { selected: LatLon | null; onSelect: (p: LatLon) => void; onMapReady?: (map: maplibregl.Map) => void })`, `interface LatLon { lat: number; lon: number }` (`web/src/map/MapView.tsx`)
  - `web/src/format.ts`: `windDir16(deg: number): string`, `weatherText(sky: number | null, pty: number | null): string`, `weatherIcon(text: string | null): string`, `formatFrameTime(iso: string): string`, `formatHour(iso: string): string`, `formatDay(date: string): string`, `formatTemp(n: number | null): string`
  - `styles.css` 클래스: `.app .map .wind-canvas .locate-btn .time-slider(__body, __label) .badge .wind-status .toast .panel(__header, __msg, __msg--error, __notice) .current(__temp, __meta) .hourly(__item) .daily .num .muted .icon .temp`

- [ ] **Step 1: web 패키지와 의존성**

`web/package.json`:

```json
{
  "name": "web",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc -p tsconfig.json && vite build",
    "test": "vitest run",
    "typecheck": "tsc -p tsconfig.json"
  }
}
```

Run:

```bash
pnpm --filter web add react@^19.3.1 react-dom@^19.3.1 maplibre-gl@^6.16.0
pnpm --filter web add -D vite@^8.3.5 @vitejs/plugin-react@^6.1.2 typescript@^5.9.3 vitest@^5.0.5 @types/react@^19.3.0 @types/react-dom@^19.3.0
```

`web/tsconfig.json`:

```json
{
  "extends": "../tsconfig.base.json",
  "compilerOptions": {
    "lib": ["ES2023", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "jsx": "react-jsx",
    "noEmit": true,
    "allowImportingTsExtensions": true,
    "types": ["vite/client"]
  },
  "include": ["src", "vite.config.ts"]
}
```

`web/vite.config.ts`:

```ts
import react from '@vitejs/plugin-react'
import { defineConfig } from 'vitest/config'

export default defineConfig({
  base: '/weather/',
  plugins: [react()],
  server: {
    // 개발 중에는 Hono 서버(3090)로 넘긴다
    proxy: {
      '/weather/api': 'http://127.0.0.1:3090',
      '/weather/data': 'http://127.0.0.1:3090',
    },
  },
  test: { environment: 'node', include: ['src/**/*.test.ts'] },
})
```

`web/index.html`:

```html
<!doctype html>
<html lang="ko">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
    <meta name="description" content="동아시아 바람 지도와 기상청 상세 예보" />
    <title>바람 지도</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

- [ ] **Step 2: format 실패 테스트 작성**

`web/src/format.test.ts`:

```ts
import { describe, expect, it } from 'vitest'
import { formatDay, formatFrameTime, formatHour, formatTemp, weatherIcon, weatherText, windDir16 } from './format'

describe('windDir16', () => {
  it('가이드 변환식', () => {
    expect(windDir16(0)).toBe('북')
    expect(windDir16(339)).toBe('북북서')
    expect(windDir16(165)).toBe('남남동')
    expect(windDir16(359)).toBe('북')
    expect(windDir16(90)).toBe('동')
  })
})

describe('weatherText / weatherIcon', () => {
  it('강수형태가 있으면 강수형태', () => {
    expect(weatherText(1, 0)).toBe('맑음')
    expect(weatherText(4, 1)).toBe('비')
    expect(weatherText(3, 7)).toBe('눈날림')
    expect(weatherText(null, null)).toBe('-')
  })

  it('문구에 맞는 아이콘', () => {
    expect(weatherIcon('맑음')).toBe('☀️')
    expect(weatherIcon('구름많음')).toBe('⛅')
    expect(weatherIcon('흐림')).toBe('☁️')
    expect(weatherIcon('흐리고 비')).toBe('🌧️')
    expect(weatherIcon('구름많고 눈')).toBe('🌨️')
    expect(weatherIcon('구름많고 소나기')).toBe('🌦️')
    expect(weatherIcon(null)).toBe('·')
  })
})

describe('시각 포맷 (KST)', () => {
  it('바람 프레임 시각(UTC)을 KST로', () => {
    expect(formatFrameTime('2026-10-27T06:00:00Z')).toBe('10/27(화) 15시')
  })

  it('시간별, 날짜, 기온', () => {
    expect(formatHour('2026-10-28T00:00:00+09:00')).toBe('0시')
    expect(formatHour('2026-10-27T15:00:00+09:00')).toBe('15시')
    expect(formatDay('2026-10-28')).toBe('10/28(수)')
    expect(formatTemp(13.6)).toBe('14°')
    expect(formatTemp(null)).toBe('-')
  })
})
```

Run: `pnpm --filter web test`
Expected: FAIL (`./format` 없음)

- [ ] **Step 3: format 구현**

`web/src/format.ts`:

```ts
// 화면 표시용 문구·시각 포맷 (시각은 모두 KST)

const DIRS = ['북', '북북동', '북동', '동북동', '동', '동남동', '남동', '남남동', '남', '남남서', '남서', '서남서', '서', '서북서', '북서', '북북서']

/** 기상청 가이드 16방위 변환식 */
export function windDir16(deg: number): string {
  return DIRS[Math.floor((deg + 22.5 * 0.5) / 22.5) % 16]!
}

const SKY: Record<number, string> = { 1: '맑음', 3: '구름많음', 4: '흐림' }
const PTY: Record<number, string> = { 1: '비', 2: '비/눈', 3: '눈', 4: '소나기', 5: '빗방울', 6: '빗방울눈날림', 7: '눈날림' }

export function weatherText(sky: number | null, pty: number | null): string {
  if (pty && PTY[pty]) return PTY[pty]
  return (sky && SKY[sky]) || '-'
}

export function weatherIcon(text: string | null): string {
  if (!text) return '·'
  if (text.includes('눈')) return '🌨️'
  if (text.includes('소나기')) return '🌦️'
  if (text.includes('비') || text.includes('빗방울')) return '🌧️'
  if (text.includes('흐')) return '☁️'
  if (text.includes('구름')) return '⛅'
  if (text.includes('맑')) return '☀️'
  return '·'
}

const WEEKDAYS = '일월화수목금토'
const KST_OFFSET = 9 * 3_600_000

function kst(iso: string): Date {
  return new Date(Date.parse(iso) + KST_OFFSET)
}

export function formatFrameTime(iso: string): string {
  const d = kst(iso)
  return `${d.getUTCMonth() + 1}/${d.getUTCDate()}(${WEEKDAYS[d.getUTCDay()]}) ${d.getUTCHours()}시`
}

export function formatHour(iso: string): string {
  return `${kst(iso).getUTCHours()}시`
}

/** 'YYYY-MM-DD' → '10/28(수)' */
export function formatDay(date: string): string {
  const d = new Date(`${date}T00:00:00Z`)
  return `${d.getUTCMonth() + 1}/${d.getUTCDate()}(${WEEKDAYS[d.getUTCDay()]})`
}

export function formatTemp(n: number | null): string {
  return n === null ? '-' : `${Math.round(n)}°`
}
```

Run: `pnpm --filter web test`
Expected: 5 passed

- [ ] **Step 4: 지도·앱 작성**

`web/src/styles.css`:

```css
:root {
  --bg: #0f1720;
  --panel: rgba(17, 24, 33, 0.92);
  --text: #e8eef5;
  --muted: #9fb0c3;
  --accent: #ff5a36;
  --border: rgba(255, 255, 255, 0.12);
  color-scheme: dark;
  font-family: system-ui, -apple-system, 'Apple SD Gothic Neo', 'Noto Sans KR', sans-serif;
}
* { box-sizing: border-box; }
html, body, #root { margin: 0; height: 100%; background: var(--bg); color: var(--text); }
.app { position: relative; height: 100%; overflow: hidden; }
.map { position: absolute; inset: 0; }
.wind-canvas { position: absolute; inset: 0; pointer-events: none; }
.locate-btn {
  position: absolute; top: 80px; right: 10px; z-index: 2; width: 29px; height: 29px;
  border: 0; border-radius: 4px; background: #fff; color: #333; font-size: 16px; cursor: pointer;
  box-shadow: 0 0 0 2px rgba(0, 0, 0, 0.1);
}
.time-slider {
  position: absolute; left: 50%; bottom: 28px; transform: translateX(-50%); z-index: 2;
  display: flex; align-items: center; gap: 8px; width: min(560px, calc(100% - 32px)); padding: 8px 12px;
  background: var(--panel); border: 1px solid var(--border); border-radius: 12px;
}
.time-slider button { padding: 0 6px; border: 0; background: none; color: var(--text); font-size: 22px; cursor: pointer; }
.time-slider button:disabled { opacity: 0.3; cursor: default; }
.time-slider__body { flex: 1; display: flex; flex-direction: column; gap: 4px; }
.time-slider__label { display: flex; align-items: center; gap: 8px; font-size: 14px; font-weight: 600; }
.time-slider input { width: 100%; accent-color: var(--accent); }
.badge { padding: 1px 6px; border-radius: 999px; background: #8a5a00; color: #ffe2a8; font-size: 11px; font-weight: 500; }
.wind-status, .toast {
  position: absolute; left: 50%; bottom: 28px; transform: translateX(-50%); z-index: 2;
  padding: 8px 14px; border-radius: 999px; background: var(--panel); color: var(--muted); font-size: 13px;
}
.toast { bottom: 90px; color: var(--text); }
.panel {
  position: absolute; z-index: 3; top: 10px; left: 10px; bottom: 10px; width: 360px; overflow-y: auto;
  padding: 14px 16px; background: var(--panel); border: 1px solid var(--border); border-radius: 14px;
  backdrop-filter: blur(6px);
}
.panel__header { display: flex; justify-content: space-between; align-items: flex-start; gap: 8px; }
.panel__header h2 { margin: 0; font-size: 16px; }
.panel__header button { border: 0; background: none; color: var(--muted); font-size: 22px; line-height: 1; cursor: pointer; }
.panel__msg { color: var(--muted); font-size: 14px; }
.panel__msg--error { color: #ff9b8a; }
.panel__notice { padding: 6px 8px; border-radius: 8px; background: rgba(138, 90, 0, 0.35); color: #ffe2a8; font-size: 12px; }
.panel section { margin-top: 16px; }
.panel h3 { margin: 0 0 8px; color: var(--muted); font-size: 13px; font-weight: 600; }
.current { display: flex; align-items: center; gap: 14px; }
.current__temp { font-size: 44px; font-weight: 700; }
.current__meta { color: var(--muted); font-size: 13px; line-height: 1.6; }
.hourly { display: flex; gap: 6px; overflow-x: auto; padding-bottom: 6px; }
.hourly__item { flex: 0 0 auto; width: 58px; padding: 6px 2px; border-radius: 8px; background: rgba(255, 255, 255, 0.05); font-size: 12px; text-align: center; }
.icon { font-size: 18px; }
.temp { font-size: 14px; font-weight: 600; }
.muted { color: var(--muted); }
.daily { width: 100%; border-collapse: collapse; font-size: 13px; }
.daily td { padding: 6px 2px; border-top: 1px solid var(--border); }
.daily td.num { text-align: right; white-space: nowrap; }
@media (max-width: 640px) {
  .panel { top: auto; left: 0; right: 0; bottom: 0; width: auto; max-height: 62%; border-radius: 16px 16px 0 0; }
}
```

`web/src/map/MapView.tsx`:

```tsx
import maplibregl from 'maplibre-gl'
import { useEffect, useRef } from 'react'

export interface LatLon {
  lat: number
  lon: number
}

interface Props {
  selected: LatLon | null
  onSelect: (p: LatLon) => void
  onMapReady?: (map: maplibregl.Map) => void
}

const STYLE_URL = 'https://tiles.openfreemap.org/styles/liberty'

export function MapView({ selected, onSelect, onMapReady }: Props) {
  const container = useRef<HTMLDivElement>(null)
  const mapRef = useRef<maplibregl.Map | null>(null)
  const markerRef = useRef<maplibregl.Marker | null>(null)
  // 지도는 한 번만 만들고, 콜백은 최신 것을 쓴다
  const onSelectRef = useRef(onSelect)
  const onReadyRef = useRef(onMapReady)
  useEffect(() => {
    onSelectRef.current = onSelect
    onReadyRef.current = onMapReady
  })

  useEffect(() => {
    const map = new maplibregl.Map({
      container: container.current!,
      style: STYLE_URL,
      center: [127.8, 36.2],
      zoom: 5,
      minZoom: 3,
      maxZoom: 12,
      attributionControl: { customAttribution: '예보: 기상청 · 바람: NOAA GFS' },
    })
    map.addControl(new maplibregl.NavigationControl({ showCompass: false }), 'top-right')
    map.on('click', (e) => onSelectRef.current({ lat: e.lngLat.lat, lon: e.lngLat.lng }))
    map.on('load', () => onReadyRef.current?.(map))
    mapRef.current = map
    return () => {
      map.remove()
      mapRef.current = null
      markerRef.current = null
    }
  }, [])

  useEffect(() => {
    const map = mapRef.current
    if (!map) return
    if (!selected) {
      markerRef.current?.remove()
      markerRef.current = null
      return
    }
    markerRef.current ??= new maplibregl.Marker({ color: '#ff5a36' })
    markerRef.current.setLngLat([selected.lon, selected.lat]).addTo(map)
  }, [selected])

  return <div ref={container} className="map" />
}
```

`web/src/App.tsx`:

```tsx
import { useState } from 'react'
import { MapView, type LatLon } from './map/MapView'

export default function App() {
  const [selected, setSelected] = useState<LatLon | null>(null)

  return (
    <div className="app">
      <MapView selected={selected} onSelect={setSelected} />
      {selected && (
        <div className="toast" role="status">
          {selected.lat.toFixed(3)}, {selected.lon.toFixed(3)}
        </div>
      )}
    </div>
  )
}
```

`web/src/main.tsx`:

```tsx
import 'maplibre-gl/dist/maplibre-gl.css'
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import App from './App'
import './styles.css'

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <App />
  </StrictMode>,
)
```

- [ ] **Step 5: 빌드와 화면 확인**

Run: `pnpm --filter web build`
Expected: 타입 오류 없이 `web/dist/index.html`, `web/dist/assets/*` 생성 (asset 경로가 `/weather/assets/...`)

Run: `pnpm dev` (server와 web을 함께 실행) → 브라우저로 `http://localhost:5173/weather/` 열기
Expected: 한반도 중심 지도가 보이고, 클릭하면 주황 마커와 하단에 좌표가 뜬다. 확인 후 끈다.

- [ ] **Step 6: 커밋**

```bash
git add web pnpm-lock.yaml
git commit -m "feat: 웹 뼈대와 MapLibre 지도" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: 바람 입자 레이어와 시간 슬라이더

**Files:**
- Create: `web/src/wind/field.ts`, `web/src/wind/particles.ts`, `web/src/wind/WindParticles.ts`, `web/src/wind/useWind.ts`, `web/src/components/TimeSlider.tsx`
- Modify: `web/src/App.tsx`
- Test: `web/src/wind/field.test.ts`, `web/src/wind/particles.test.ts`

**Interfaces:**
- Consumes: `latest.json` 형식(Task 9), `formatFrameTime`(Task 10), `MapView`의 `onMapReady`
- Produces:
  - `field.ts`: `WindFrame`, `WindMeta`, `WindField { width; height; bbox: [number, number, number, number]; u: Float32Array; v: Float32Array }`, `decodeField(rgba, meta, frame)`, `sampleWind(f, lon, lat): [number, number] | null`, `pickFrameIndex(frames, now: Date): number`, `speedColor(speed): [number, number, number]`, `speedImage(f, outHeight): { data: Uint8ClampedArray<ArrayBuffer>; width: number; height: number }`
  - `particles.ts`: `Particle { lon; lat; age }`, `Bounds { west; south; east; north }`, `randomParticle(b, maxAge, rand?)`, `stepParticle(p, wind, degPerMs): boolean`, `degPerMsForZoom(zoom): number`, `speedBucket(speed): number` (0~5)
  - `useWind(map): { meta: WindMeta | null; status: 'loading' | 'ready' | 'missing'; index: number; setIndex: (i: number) => void; stale: boolean }`
  - `TimeSlider(props: { frames: WindFrame[]; index: number; onChange: (i: number) => void; stale: boolean })`

- [ ] **Step 1: 실패 테스트 작성**

`web/src/wind/field.test.ts`:

```ts
import { describe, expect, it } from 'vitest'
import { decodeField, pickFrameIndex, sampleWind, speedColor, speedImage, type WindField, type WindFrame } from './field'

const frame = (time: string): WindFrame => ({ time, file: 'x.png', uMin: -10, uMax: 10, vMin: 0, vMax: 20 })

// 3×2 격자, bbox 110~150°E, 20~50°N. 행 0이 북쪽
function field(u: number[], v: number[]): WindField {
  return { width: 3, height: 2, bbox: [110, 20, 150, 50], u: Float32Array.from(u), v: Float32Array.from(v) }
}

describe('decodeField', () => {
  it('R/G를 min~max로 되돌린다', () => {
    const rgba = new Uint8ClampedArray([0, 255, 0, 255, 255, 0, 0, 255])
    const f = decodeField(rgba, { width: 2, height: 1, bbox: [0, 0, 1, 1] }, frame('2026-10-27T06:00:00Z'))
    expect(Array.from(f.u)).toEqual([-10, 10])
    expect(Array.from(f.v)).toEqual([20, 0])
  })
})

describe('sampleWind', () => {
  const f = field([10, 0, 0, 0, 0, 0], [0, 0, 0, 0, 0, 4])

  it('격자점은 그 값, 북서 모서리는 행 0 열 0', () => {
    expect(sampleWind(f, 110, 50)).toEqual([10, 0])
    expect(sampleWind(f, 150, 20)).toEqual([0, 4])
  })

  it('사이는 쌍선형 보간', () => {
    expect(sampleWind(f, 120, 50)?.[0]).toBeCloseTo(5)
    expect(sampleWind(f, 110, 35)?.[0]).toBeCloseTo(5)
  })

  it('범위 밖은 null', () => {
    expect(sampleWind(f, 109.9, 30)).toBeNull()
    expect(sampleWind(f, 130, 50.1)).toBeNull()
  })
})

describe('pickFrameIndex', () => {
  it('현재와 가장 가까운 프레임', () => {
    const frames = [frame('2026-10-27T06:00:00Z'), frame('2026-10-27T09:00:00Z'), frame('2026-10-27T12:00:00Z')]
    expect(pickFrameIndex(frames, new Date('2026-10-27T10:20:00Z'))).toBe(1)
    expect(pickFrameIndex(frames, new Date('2026-10-28T00:00:00Z'))).toBe(2)
    expect(pickFrameIndex(frames, new Date('2026-10-26T00:00:00Z'))).toBe(0)
  })
})

describe('speedColor', () => {
  it('구간 사이를 보간한다', () => {
    expect(speedColor(0)).toEqual([98, 113, 183])
    expect(speedColor(1.5)).toEqual([78, 105, 171])
    expect(speedColor(100)).toEqual([175, 80, 136])
  })
})

describe('speedImage', () => {
  it('메르카토르 행 간격으로 다시 샘플링 (행 0이 북쪽)', () => {
    // 북쪽 행만 바람이 강하다
    const f = field([30, 30, 30, 0, 0, 0], [0, 0, 0, 0, 0, 0])
    const img = speedImage(f, 4)
    expect(img.width).toBe(3)
    expect(img.height).toBe(4)
    expect(img.data).toHaveLength(3 * 4 * 4)
    const red = (row: number) => img.data[row * 3 * 4]!
    expect(red(0)).toBe(speedColor(30)[0])
    expect(red(3)).toBe(speedColor(0)[0])
    expect(img.data[3]).toBe(200) // 알파
  })
})
```

`web/src/wind/particles.test.ts`:

```ts
import { describe, expect, it } from 'vitest'
import { degPerMsForZoom, randomParticle, speedBucket, stepParticle } from './particles'

const bounds = { west: 120, south: 30, east: 130, north: 40 }

describe('randomParticle', () => {
  it('범위 안에 만든다', () => {
    const p = randomParticle(bounds, 90, () => 0.5)
    expect(p.lon).toBe(125)
    expect(p.lat).toBe(35)
    expect(p.age).toBeGreaterThan(0)
    expect(p.age).toBeLessThanOrEqual(90)
  })
})

describe('stepParticle', () => {
  it('u는 동쪽, v는 북쪽으로 움직인다', () => {
    const p = { lon: 125, lat: 0, age: 10 }
    expect(stepParticle(p, [10, 5], 0.01)).toBe(true)
    expect(p.lon).toBeCloseTo(125.1)
    expect(p.lat).toBeCloseTo(0.05)
    expect(p.age).toBe(9)
  })

  it('바람이 없거나 수명이 다하면 false', () => {
    expect(stepParticle({ lon: 0, lat: 0, age: 10 }, null, 0.01)).toBe(false)
    expect(stepParticle({ lon: 0, lat: 0, age: 1 }, [1, 1], 0.01)).toBe(false)
  })
})

describe('degPerMsForZoom', () => {
  it('줌이 1 오르면 절반', () => {
    expect(degPerMsForZoom(6)).toBeCloseTo(degPerMsForZoom(5) / 2)
  })
})

describe('speedBucket', () => {
  it('풍속 구간', () => {
    expect(speedBucket(0)).toBe(0)
    expect(speedBucket(5)).toBe(1)
    expect(speedBucket(12)).toBe(3)
    expect(speedBucket(40)).toBe(5)
  })
})
```

Run: `pnpm --filter web test`
Expected: FAIL (`./field`, `./particles` 없음)

- [ ] **Step 2: field·particles 구현**

`web/src/wind/field.ts`:

```ts
// 바람 PNG ↔ 바람장, 샘플링, 프레임 선택, 풍속 배경 이미지 (순수 함수)

export interface WindFrame {
  /** UTC ISO */
  time: string
  /** latest.json 기준 상대 경로 "2026102706/f000.png" */
  file: string
  uMin: number
  uMax: number
  vMin: number
  vMax: number
}

export interface WindMeta {
  cycle: string
  /** [west, south, east, north] */
  bbox: [number, number, number, number]
  width: number
  height: number
  frames: WindFrame[]
}

export interface WindField {
  width: number
  height: number
  bbox: [number, number, number, number]
  /** 행 0이 북쪽, m/s */
  u: Float32Array
  v: Float32Array
}

export function decodeField(rgba: Uint8ClampedArray, meta: Pick<WindMeta, 'width' | 'height' | 'bbox'>, frame: WindFrame): WindField {
  const n = meta.width * meta.height
  const u = new Float32Array(n)
  const v = new Float32Array(n)
  const uSpan = frame.uMax - frame.uMin
  const vSpan = frame.vMax - frame.vMin
  for (let i = 0; i < n; i++) {
    u[i] = frame.uMin + (rgba[i * 4]! / 255) * uSpan
    v[i] = frame.vMin + (rgba[i * 4 + 1]! / 255) * vSpan
  }
  return { width: meta.width, height: meta.height, bbox: meta.bbox, u, v }
}

export function sampleWind(f: WindField, lon: number, lat: number): [number, number] | null {
  const [west, south, east, north] = f.bbox
  if (lon < west || lon > east || lat < south || lat > north) return null
  const x = ((lon - west) / (east - west)) * (f.width - 1)
  const y = ((north - lat) / (north - south)) * (f.height - 1)
  const x0 = Math.floor(x)
  const y0 = Math.floor(y)
  const x1 = Math.min(x0 + 1, f.width - 1)
  const y1 = Math.min(y0 + 1, f.height - 1)
  const tx = x - x0
  const ty = y - y0
  const lerp = (a: Float32Array) => {
    const top = a[y0 * f.width + x0]! * (1 - tx) + a[y0 * f.width + x1]! * tx
    const bottom = a[y1 * f.width + x0]! * (1 - tx) + a[y1 * f.width + x1]! * tx
    return top * (1 - ty) + bottom * ty
  }
  return [lerp(f.u), lerp(f.v)]
}

export function pickFrameIndex(frames: WindFrame[], now: Date): number {
  let best = 0
  let bestDiff = Infinity
  frames.forEach((frame, i) => {
    const diff = Math.abs(Date.parse(frame.time) - now.getTime())
    if (diff < bestDiff) {
      best = i
      bestDiff = diff
    }
  })
  return best
}

// [풍속 m/s, RGB]
const STOPS: [number, [number, number, number]][] = [
  [0, [98, 113, 183]],
  [3, [57, 97, 159]],
  [6, [74, 148, 169]],
  [9, [77, 141, 123]],
  [12, [83, 165, 83]],
  [15, [53, 159, 53]],
  [18, [167, 157, 81]],
  [21, [159, 127, 58]],
  [25, [161, 108, 92]],
  [30, [129, 58, 78]],
  [40, [175, 80, 136]],
]

export function speedColor(speed: number): [number, number, number] {
  if (speed <= STOPS[0]![0]) return STOPS[0]![1]
  for (let i = 1; i < STOPS.length; i++) {
    const [s1, c1] = STOPS[i]!
    if (speed <= s1) {
      const [s0, c0] = STOPS[i - 1]!
      const t = (speed - s0) / (s1 - s0)
      return [0, 1, 2].map((k) => Math.round(c0[k]! + (c1[k]! - c0[k]!) * t)) as [number, number, number]
    }
  }
  return STOPS[STOPS.length - 1]![1]
}

const RAD = Math.PI / 180
const mercY = (lat: number) => Math.log(Math.tan(Math.PI / 4 + (lat * RAD) / 2))
const latFromMercY = (y: number) => (2 * Math.atan(Math.exp(y)) - Math.PI / 2) / RAD

/**
 * MapLibre image source용 풍속 색상 이미지.
 * image source는 네 모서리 사이를 메르카토르 공간에서 선형으로 펴므로, 행을 메르카토르 y 등간격으로 다시 샘플링한다
 */
export function speedImage(f: WindField, outHeight: number): { data: Uint8ClampedArray<ArrayBuffer>; width: number; height: number } {
  const [west, south, east, north] = f.bbox
  const width = f.width
  const data = new Uint8ClampedArray(width * outHeight * 4)
  const yNorth = mercY(north)
  const ySouth = mercY(south)
  for (let row = 0; row < outHeight; row++) {
    const lat = latFromMercY(yNorth + ((ySouth - yNorth) * row) / (outHeight - 1))
    for (let col = 0; col < width; col++) {
      const lon = west + ((east - west) * col) / (width - 1)
      const wind = sampleWind(f, lon, Math.min(north, Math.max(south, lat)))
      const [r, g, b] = speedColor(wind ? Math.hypot(wind[0], wind[1]) : 0)
      const i = (row * width + col) * 4
      data[i] = r
      data[i + 1] = g
      data[i + 2] = b
      data[i + 3] = 200
    }
  }
  return { data, width, height: outHeight }
}
```

`web/src/wind/particles.ts`:

```ts
// 바람 입자 생성·이동 (순수 함수)

export interface Particle {
  lon: number
  lat: number
  /** 남은 프레임 수 */
  age: number
}

export interface Bounds {
  west: number
  south: number
  east: number
  north: number
}

export function randomParticle(b: Bounds, maxAge: number, rand: () => number = Math.random): Particle {
  return {
    lon: b.west + (b.east - b.west) * rand(),
    lat: b.south + (b.north - b.south) * rand(),
    age: 1 + Math.floor(rand() * maxAge),
  }
}

/** 한 프레임 이동. 바람이 없거나 수명이 다하면 false (다시 뿌려야 함) */
export function stepParticle(p: Particle, wind: [number, number] | null, degPerMs: number): boolean {
  if (!wind) return false
  p.lon += (wind[0] * degPerMs) / Math.cos((p.lat * Math.PI) / 180)
  p.lat += wind[1] * degPerMs
  p.age -= 1
  return p.age > 0
}

/** 줌 5에서 10m/s가 프레임당 약 1.5px 움직이도록 (줌이 오르면 같은 화면 속도를 위해 절반) */
export function degPerMsForZoom(zoom: number): number {
  return 0.0033 * Math.pow(2, 5 - zoom)
}

const BUCKETS = [3, 6, 10, 15, 20]

export function speedBucket(speed: number): number {
  const i = BUCKETS.findIndex((limit) => speed < limit)
  return i === -1 ? BUCKETS.length : i
}
```

Run: `pnpm --filter web test`
Expected: 모두 통과

- [ ] **Step 3: 애니메이션·로딩·슬라이더 작성**

`web/src/wind/WindParticles.ts`:

```ts
// 지도 위 Canvas 2D 바람 입자 애니메이션
import type { Map as MlMap } from 'maplibre-gl'
import { sampleWind, type WindField } from './field'
import { degPerMsForZoom, randomParticle, speedBucket, stepParticle, type Bounds, type Particle } from './particles'

const BUCKET_COLORS = [
  'rgba(255,255,255,0.45)',
  'rgba(255,255,255,0.6)',
  'rgba(255,255,255,0.75)',
  'rgba(255,250,225,0.85)',
  'rgba(255,235,190,0.92)',
  'rgba(255,215,165,1)',
]
const FADE_ALPHA = 0.92 // 프레임마다 이전 그림을 이만큼 남긴다
const MAX_AGE = 90
const PIXELS_PER_PARTICLE = 450
const MAX_PARTICLES = 6000

export class WindParticles {
  private readonly canvas = document.createElement('canvas')
  private readonly ctx: CanvasRenderingContext2D
  private particles: Particle[] = []
  private field: WindField | null = null
  private moving = false
  private raf = 0

  constructor(private readonly map: MlMap) {
    this.canvas.className = 'wind-canvas'
    map.getCanvasContainer().appendChild(this.canvas)
    this.ctx = this.canvas.getContext('2d')!
    map.on('movestart', this.onMoveStart)
    map.on('moveend', this.reset)
    map.on('resize', this.reset)
    this.reset()
    this.raf = requestAnimationFrame(this.loop)
  }

  setField(field: WindField | null): void {
    this.field = field
    this.reset()
  }

  destroy(): void {
    cancelAnimationFrame(this.raf)
    this.map.off('movestart', this.onMoveStart)
    this.map.off('moveend', this.reset)
    this.map.off('resize', this.reset)
    this.canvas.remove()
  }

  private get size() {
    const el = this.map.getContainer()
    return { width: el.clientWidth, height: el.clientHeight }
  }

  // 지도가 움직이는 동안은 입자를 숨긴다 (화면 좌표가 맞지 않음)
  private onMoveStart = () => {
    this.moving = true
    this.clear()
  }

  private reset = () => {
    this.moving = false
    const { width, height } = this.size
    const dpr = window.devicePixelRatio || 1
    this.canvas.width = width * dpr
    this.canvas.height = height * dpr
    this.canvas.style.width = `${width}px`
    this.canvas.style.height = `${height}px`
    this.ctx.setTransform(dpr, 0, 0, dpr, 0, 0)
    this.clear()
    const bounds = this.visibleBounds()
    const count = Math.min(MAX_PARTICLES, Math.round((width * height) / PIXELS_PER_PARTICLE))
    this.particles = bounds ? Array.from({ length: count }, () => randomParticle(bounds, MAX_AGE)) : []
  }

  private visibleBounds(): Bounds | null {
    if (!this.field) return null
    const [west, south, east, north] = this.field.bbox
    const view = this.map.getBounds()
    const b = {
      west: Math.max(west, view.getWest()),
      south: Math.max(south, view.getSouth()),
      east: Math.min(east, view.getEast()),
      north: Math.min(north, view.getNorth()),
    }
    return b.west < b.east && b.south < b.north ? b : null
  }

  private clear(): void {
    const { width, height } = this.size
    this.ctx.clearRect(0, 0, width, height)
  }

  private loop = () => {
    this.raf = requestAnimationFrame(this.loop)
    const field = this.field
    if (this.moving || !field || this.particles.length === 0) return
    const bounds = this.visibleBounds()
    if (!bounds) return
    const { ctx } = this
    const { width, height } = this.size

    ctx.globalCompositeOperation = 'destination-in'
    ctx.fillStyle = `rgba(0,0,0,${FADE_ALPHA})`
    ctx.fillRect(0, 0, width, height)
    ctx.globalCompositeOperation = 'source-over'

    const degPerMs = degPerMsForZoom(this.map.getZoom())
    const paths = BUCKET_COLORS.map(() => new Path2D())
    for (const p of this.particles) {
      const wind = sampleWind(field, p.lon, p.lat)
      const from = this.map.project([p.lon, p.lat])
      if (!wind || !stepParticle(p, wind, degPerMs)) {
        Object.assign(p, randomParticle(bounds, MAX_AGE))
        continue
      }
      const to = this.map.project([p.lon, p.lat])
      const path = paths[speedBucket(Math.hypot(wind[0], wind[1]))]!
      path.moveTo(from.x, from.y)
      path.lineTo(to.x, to.y)
    }
    ctx.lineWidth = 1.2
    paths.forEach((path, i) => {
      ctx.strokeStyle = BUCKET_COLORS[i]!
      ctx.stroke(path)
    })
  }
}
```

`web/src/wind/useWind.ts`:

```ts
import type { ImageSource, Map as MlMap } from 'maplibre-gl'
import { useEffect, useRef, useState } from 'react'
import { decodeField, pickFrameIndex, speedImage, type WindField, type WindFrame, type WindMeta } from './field'
import { WindParticles } from './WindParticles'

const BASE = `${import.meta.env.BASE_URL}data/wind/`
const SPEED_LAYER = 'wind-speed'
const STALE_HOURS = 14

async function loadLatest(): Promise<WindMeta | null> {
  try {
    const res = await fetch(`${BASE}latest.json`, { cache: 'no-cache' })
    if (!res.ok) return null
    const meta = (await res.json()) as WindMeta
    return meta.frames?.length ? meta : null
  } catch {
    return null
  }
}

async function loadFrameField(meta: WindMeta, frame: WindFrame): Promise<WindField> {
  const img = new Image()
  img.src = BASE + frame.file
  await img.decode()
  const canvas = document.createElement('canvas')
  canvas.width = meta.width
  canvas.height = meta.height
  const ctx = canvas.getContext('2d', { willReadFrequently: true })!
  ctx.drawImage(img, 0, 0)
  return decodeField(ctx.getImageData(0, 0, meta.width, meta.height).data, meta, frame)
}

// 풍속 배경을 라벨(symbol) 레이어 아래에 깐다
function setSpeedLayer(map: MlMap, field: WindField): void {
  const img = speedImage(field, 512)
  const canvas = document.createElement('canvas')
  canvas.width = img.width
  canvas.height = img.height
  canvas.getContext('2d')!.putImageData(new ImageData(img.data, img.width, img.height), 0, 0)
  const url = canvas.toDataURL()
  const [west, south, east, north] = field.bbox
  const coordinates: [[number, number], [number, number], [number, number], [number, number]] = [
    [west, north],
    [east, north],
    [east, south],
    [west, south],
  ]
  const source = map.getSource(SPEED_LAYER) as ImageSource | undefined
  if (source) {
    source.updateImage({ url, coordinates })
    return
  }
  map.addSource(SPEED_LAYER, { type: 'image', url, coordinates })
  const firstSymbol = map.getStyle().layers.find((l) => l.type === 'symbol')?.id
  map.addLayer({ id: SPEED_LAYER, type: 'raster', source: SPEED_LAYER, paint: { 'raster-opacity': 0.55 } }, firstSymbol)
}

export function useWind(map: MlMap | null) {
  const [meta, setMeta] = useState<WindMeta | null>(null)
  const [status, setStatus] = useState<'loading' | 'ready' | 'missing'>('loading')
  const [index, setIndex] = useState(0)
  const particles = useRef<WindParticles | null>(null)

  useEffect(() => {
    loadLatest().then((m) => {
      if (!m) {
        setStatus('missing')
        return
      }
      setMeta(m)
      setIndex(pickFrameIndex(m.frames, new Date()))
      setStatus('ready')
    })
  }, [])

  useEffect(() => {
    if (!map) return
    const p = new WindParticles(map)
    particles.current = p
    return () => {
      p.destroy()
      particles.current = null
    }
  }, [map])

  useEffect(() => {
    const frame = meta?.frames[index]
    if (!map || !meta || !frame) return
    let cancelled = false
    loadFrameField(meta, frame)
      .then((field) => {
        if (cancelled) return
        particles.current?.setField(field)
        setSpeedLayer(map, field)
      })
      .catch(() => {
        if (!cancelled) setStatus('missing')
      })
    return () => {
      cancelled = true
    }
  }, [map, meta, index])

  const cycleTime = meta?.frames[0] ? Date.parse(meta.frames[0].time) : Date.now()
  const stale = (Date.now() - cycleTime) / 3_600_000 > STALE_HOURS
  return { meta, status, index, setIndex, stale }
}
```

`web/src/components/TimeSlider.tsx`:

```tsx
import { formatFrameTime } from '../format'
import type { WindFrame } from '../wind/field'

interface Props {
  frames: WindFrame[]
  index: number
  onChange: (i: number) => void
  stale: boolean
}

export function TimeSlider({ frames, index, onChange, stale }: Props) {
  const frame = frames[index]
  if (!frame) return null
  return (
    <div className="time-slider">
      <button type="button" onClick={() => onChange(index - 1)} disabled={index === 0} aria-label="이전 시각">
        ‹
      </button>
      <div className="time-slider__body">
        <div className="time-slider__label">
          {formatFrameTime(frame.time)} 바람
          {stale && <span className="badge">업데이트 지연</span>}
        </div>
        <input
          type="range"
          min={0}
          max={frames.length - 1}
          value={index}
          onChange={(e) => onChange(Number(e.target.value))}
          aria-label="예보 시각"
        />
      </div>
      <button type="button" onClick={() => onChange(index + 1)} disabled={index === frames.length - 1} aria-label="다음 시각">
        ›
      </button>
    </div>
  )
}
```

`web/src/App.tsx` 전체를 아래로 바꾼다:

```tsx
import type { Map as MlMap } from 'maplibre-gl'
import { useState } from 'react'
import { TimeSlider } from './components/TimeSlider'
import { MapView, type LatLon } from './map/MapView'
import { useWind } from './wind/useWind'

export default function App() {
  const [map, setMap] = useState<MlMap | null>(null)
  const [selected, setSelected] = useState<LatLon | null>(null)
  const wind = useWind(map)

  return (
    <div className="app">
      <MapView selected={selected} onSelect={setSelected} onMapReady={setMap} />
      {wind.status === 'ready' && wind.meta && (
        <TimeSlider frames={wind.meta.frames} index={wind.index} onChange={wind.setIndex} stale={wind.stale} />
      )}
      {wind.status === 'missing' && <div className="wind-status">바람 데이터 준비 중</div>}
    </div>
  )
}
```

- [ ] **Step 4: 테스트·빌드·화면 확인**

Run: `pnpm --filter web test && pnpm --filter web build`
Expected: 테스트 통과, 빌드 성공

Run: `pnpm dev` → `http://localhost:5173/weather/` (Task 9 Step 6에서 만든 `data/wind/`가 있어야 함)
Expected: 동아시아 영역에 풍속 색 배경과 흰 입자 애니메이션이 흐른다. 지도를 움직이면 입자가 사라졌다가 멈추면 다시 나온다. 슬라이더로 시각을 바꾸면 배경·입자가 바뀐다. `data/wind`를 잠시 다른 이름으로 바꾸고 새로고침하면 "바람 데이터 준비 중"이 뜬다(확인 후 원래 이름으로).

- [ ] **Step 5: 커밋**

```bash
git add web/src
git commit -m "feat: 바람 입자 애니메이션, 풍속 배경, 시간 슬라이더" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: 예보 패널과 내 위치

**Files:**
- Create: `web/src/api.ts`, `web/src/components/ForecastPanel.tsx`
- Modify: `web/src/App.tsx`

**Interfaces:**
- Consumes: `ForecastResponse` 등(`server/src/types.ts`), `format.ts` 함수들, `LatLon`
- Produces: `fetchForecast(lat: number, lon: number, signal?: AbortSignal): Promise<ForecastResponse>`, `ForecastPanel(props: { point: LatLon; onClose: () => void })`

- [ ] **Step 1: API 호출 작성**

`web/src/api.ts`:

```ts
import type { ForecastResponse } from '../../server/src/types.ts'

export type { Current, Daily, ForecastResponse, Hourly } from '../../server/src/types.ts'

export async function fetchForecast(lat: number, lon: number, signal?: AbortSignal): Promise<ForecastResponse> {
  const url = `${import.meta.env.BASE_URL}api/forecast?lat=${lat.toFixed(4)}&lon=${lon.toFixed(4)}`
  const res = await fetch(url, { signal })
  if (!res.ok) {
    const body = (await res.json().catch(() => null)) as { error?: string } | null
    throw new Error(body?.error ?? `예보를 불러오지 못했습니다 (${res.status})`)
  }
  return (await res.json()) as ForecastResponse
}
```

- [ ] **Step 2: 패널 작성**

`web/src/components/ForecastPanel.tsx`:

```tsx
import { useEffect, useState } from 'react'
import { fetchForecast, type Current, type Daily, type ForecastResponse, type Hourly } from '../api'
import { formatDay, formatHour, formatTemp, weatherIcon, weatherText, windDir16 } from '../format'
import type { LatLon } from '../map/MapView'

type State = { status: 'loading' } | { status: 'error'; message: string } | { status: 'ok'; data: ForecastResponse }

interface Props {
  point: LatLon
  onClose: () => void
}

export function ForecastPanel({ point, onClose }: Props) {
  const [state, setState] = useState<State>({ status: 'loading' })

  useEffect(() => {
    const ctrl = new AbortController()
    setState({ status: 'loading' })
    fetchForecast(point.lat, point.lon, ctrl.signal)
      .then((data) => setState({ status: 'ok', data }))
      .catch((err: Error) => {
        if (!ctrl.signal.aborted) setState({ status: 'error', message: err.message })
      })
    return () => ctrl.abort()
  }, [point.lat, point.lon])

  const title = state.status === 'ok' && state.data.location ? state.data.location.name : '선택한 위치'

  return (
    <aside className="panel" aria-label="상세 예보">
      <header className="panel__header">
        <h2>{title}</h2>
        <button type="button" onClick={onClose} aria-label="닫기">
          ×
        </button>
      </header>
      {state.status === 'loading' && <p className="panel__msg">불러오는 중…</p>}
      {state.status === 'error' && <p className="panel__msg panel__msg--error">{state.message}</p>}
      {state.status === 'ok' && <ForecastBody data={state.data} />}
    </aside>
  )
}

function ForecastBody({ data }: { data: ForecastResponse }) {
  if (!data.location) return <p className="panel__msg">국내(남한)만 상세 예보를 지원합니다.</p>
  return (
    <>
      {data.stale && <p className="panel__notice">기상청 응답이 늦어 이전 자료를 보여주고 있습니다.</p>}
      <CurrentSection current={data.current} sky={data.hourly[0]?.sky ?? null} error={data.errors.current} />
      <HourlySection hourly={data.hourly} error={data.errors.hourly} />
      <DailySection daily={data.daily} error={data.errors.daily} />
      <p className="panel__msg">출처: 기상청</p>
    </>
  )
}

function wind(dir: number | null, speed: number | null, text: string | null = null): string {
  const from = dir === null ? '' : `${windDir16(dir)}풍 `
  if (speed !== null) return `${from}${speed}m/s`
  return text ? `${from}${text}` : '-'
}

function CurrentSection({ current, sky, error }: { current: Current | null; sky: number | null; error?: string }) {
  if (error) return <p className="panel__msg panel__msg--error">현재 날씨: {error}</p>
  if (!current) return null
  const text = weatherText(sky, current.pty)
  return (
    <section className="current">
      <div className="current__temp">{formatTemp(current.temp)}</div>
      <div className="current__meta">
        <div>
          {weatherIcon(text)} {text}
        </div>
        <div>습도 {current.humidity ?? '-'}%</div>
        <div>바람 {wind(current.windDir, current.windSpeed)}</div>
        {current.rain1h ? <div>1시간 강수 {current.rain1h}mm</div> : null}
      </div>
    </section>
  )
}

function HourlySection({ hourly, error }: { hourly: Hourly[]; error?: string }) {
  if (error) return <p className="panel__msg panel__msg--error">시간별 예보: {error}</p>
  const from = Date.now() - 3_600_000
  const upcoming = hourly.filter((h) => Date.parse(h.time) >= from)
  if (upcoming.length === 0) return null
  return (
    <section>
      <h3>시간별</h3>
      <div className="hourly">
        {upcoming.map((h) => {
          const text = weatherText(h.sky, h.pty)
          return (
            <div key={h.time} className="hourly__item" title={text}>
              <div className="muted">{formatHour(h.time)}</div>
              <div className="icon">{weatherIcon(text)}</div>
              <div className="temp">{formatTemp(h.temp)}</div>
              <div className="muted">{h.pop ?? '-'}%</div>
              <div className="muted">{h.windSpeed !== null ? `${h.windSpeed}m/s` : (h.windText ?? '-')}</div>
              {h.pcp && <div>{h.pcp}</div>}
              {h.sno && <div>{h.sno}</div>}
            </div>
          )
        })}
      </div>
    </section>
  )
}

function DailySection({ daily, error }: { daily: Daily[]; error?: string }) {
  return (
    <section>
      <h3>10일</h3>
      {error && <p className="panel__msg panel__msg--error">{error}</p>}
      <table className="daily">
        <tbody>
          {daily.map((d) => (
            <tr key={d.date}>
              <td>{formatDay(d.date)}</td>
              <td title={`오전 ${d.amWf ?? '-'} / 오후 ${d.pmWf ?? '-'}`}>
                {weatherIcon(d.amWf)} {weatherIcon(d.pmWf)}
              </td>
              <td className="num muted">
                {d.amPop ?? '-'}% / {d.pmPop ?? '-'}%
              </td>
              <td className="num">
                <span className="muted">{formatTemp(d.min)}</span> {formatTemp(d.max)}
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </section>
  )
}
```

- [ ] **Step 3: App에 패널과 내 위치 연결**

`web/src/App.tsx` 전체를 아래로 바꾼다:

```tsx
import type { Map as MlMap } from 'maplibre-gl'
import { useEffect, useState } from 'react'
import { ForecastPanel } from './components/ForecastPanel'
import { TimeSlider } from './components/TimeSlider'
import { MapView, type LatLon } from './map/MapView'
import { useWind } from './wind/useWind'

export default function App() {
  const [map, setMap] = useState<MlMap | null>(null)
  const [selected, setSelected] = useState<LatLon | null>(null)
  const [toast, setToast] = useState<string | null>(null)
  const wind = useWind(map)

  useEffect(() => {
    if (!toast) return
    const timer = setTimeout(() => setToast(null), 3000)
    return () => clearTimeout(timer)
  }, [toast])

  const locate = () => {
    if (!navigator.geolocation) {
      setToast('이 브라우저는 위치 기능을 지원하지 않습니다')
      return
    }
    navigator.geolocation.getCurrentPosition(
      (pos) => {
        const p = { lat: pos.coords.latitude, lon: pos.coords.longitude }
        setSelected(p)
        map?.flyTo({ center: [p.lon, p.lat], zoom: Math.max(map.getZoom(), 8) })
      },
      () => setToast('위치를 가져오지 못했습니다'),
      { enableHighAccuracy: false, timeout: 10_000 },
    )
  }

  return (
    <div className="app">
      <MapView selected={selected} onSelect={setSelected} onMapReady={setMap} />
      <button type="button" className="locate-btn" onClick={locate} aria-label="내 위치">
        ◎
      </button>
      {wind.status === 'ready' && wind.meta && (
        <TimeSlider frames={wind.meta.frames} index={wind.index} onChange={wind.setIndex} stale={wind.stale} />
      )}
      {wind.status === 'missing' && <div className="wind-status">바람 데이터 준비 중</div>}
      {selected && <ForecastPanel point={selected} onClose={() => setSelected(null)} />}
      {toast && (
        <div className="toast" role="status">
          {toast}
        </div>
      )}
    </div>
  )
}
```

- [ ] **Step 4: 타입체크·빌드·화면 확인**

Run: `pnpm typecheck && pnpm test && pnpm build`
Expected: server·web 모두 통과, 빌드 성공

Run: `pnpm dev` → `http://localhost:5173/weather/`
Expected:
- 서울을 클릭하면 패널에 "서울특별시 …", 현재 기온, 시간별 목록(가로 스크롤), 10일 표가 나온다.
- 베이징 근처를 클릭하면 "국내(남한)만 상세 예보를 지원합니다."가 나온다.
- ◎ 버튼은 위치 권한을 허용하면 현재 위치로 이동해 패널을 연다.
- 브라우저 폭을 640px 이하로 줄이면 패널이 하단 시트로 바뀐다.

- [ ] **Step 5: 커밋**

```bash
git add web/src
git commit -m "feat: 지점 상세 예보 패널과 내 위치" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 13: 서버 배포 (pm2 + nginx `/weather`)

**Files:**
- Create: `ecosystem.config.cjs`, `scripts/deploy.sh`, `scripts/remote-deploy.sh`, `deploy/nginx/weather.conf`, `deploy/nginx/install.sh`
- Modify: `README.md`

**Interfaces:**
- Consumes: `pnpm build` 결과(`server/dist`, `web/dist`), `wind/wind_job.py`, `/weather/healthz`

- [ ] **Step 1: pm2 설정**

`ecosystem.config.cjs`:

```js
// pm2 설정. 서버(/home/rocky/weather-app)에서 scripts/remote-deploy.sh가 startOrReload 한다
module.exports = {
  apps: [
    {
      name: 'weather',
      script: 'server/dist/index.js',
      cwd: __dirname,
      // 예보 캐시가 프로세스 메모리에 있으므로 단일 프로세스로 실행한다
      exec_mode: 'fork',
      instances: 1,
      max_memory_restart: '300M',
      time: true,
      env: {
        NODE_ENV: 'production',
        PORT: 3090,
        WIND_DATA_DIR: 'data/wind',
      },
    },
    {
      name: 'weather-wind',
      script: 'wind/wind_job.py',
      interpreter: `${__dirname}/wind/.venv/bin/python`,
      cwd: __dirname,
      // 매시 15분에 한 번 돌고 끝난다 (새 GFS 사이클이 없으면 바로 종료)
      cron_restart: '15 * * * *',
      autorestart: false,
      time: true,
      env: {
        WIND_DATA_DIR: 'data/wind',
      },
    },
  ],
}
```

- [ ] **Step 2: 배포 스크립트 (coffee-note 방식)**

`scripts/deploy.sh`:

```bash
#!/usr/bin/env bash
# 로컬에서 실행: bash scripts/deploy.sh [branch] [ref]  (branch 기본 master, ref 기본 origin/<branch>)
# 서버에 저장소가 없으면 만든다 (.env가 먼저 있어도 되도록 clone 대신 init + remote 사용)
set -euo pipefail

BRANCH="${1:-master}"
REF="${2:-origin/$BRANCH}"
PORT="${DEPLOY_PORT:-2022}"
TARGET="${DEPLOY_USER:-rocky}@${DEPLOY_HOST:-bytebard.cloud}"
REPO_URL=https://github.com/ApostleB/weather-app.git
APP_DIR=/home/rocky/weather-app

SSH_OPTS=(-p "$PORT")
if [ -n "${DEPLOY_KEY:-}" ]; then
  SSH_OPTS+=(-i "$DEPLOY_KEY")
fi

# 원격 셸에 넘기기 전에 허용 문자만 통과시킨다 ('-'로 시작하면 git 옵션으로 해석될 수 있어 거부)
for value in "$BRANCH" "$REF"; do
  if [[ ! "$value" =~ ^[A-Za-z0-9._/][A-Za-z0-9._/-]*$ ]]; then
    echo "오류: 허용되지 않는 브랜치/ref 값입니다: $value" >&2
    exit 1
  fi
done

q_branch="$(printf '%q' "$BRANCH")"
q_ref="$(printf '%q' "$REF")"

ssh "${SSH_OPTS[@]}" "$TARGET" \
  "set -euo pipefail; if [ ! -d $APP_DIR/.git ]; then mkdir -p $APP_DIR && git init -q $APP_DIR && git -C $APP_DIR remote add origin $REPO_URL; fi; cd $APP_DIR && git fetch origin && git checkout -B $q_branch $q_ref && bash scripts/remote-deploy.sh $q_branch $q_ref"
```

`scripts/remote-deploy.sh`:

```bash
#!/usr/bin/env bash
# 서버에서 실행: bash scripts/remote-deploy.sh [branch] [ref]
# branch 기본 master, ref 기본 origin/<branch> (커밋 sha도 가능)
set -euo pipefail

BRANCH="${1:-master}"
REF="${2:-origin/$BRANCH}"
APP_DIR=/home/rocky/weather-app

export PATH="$HOME/.local/bin:$PATH"

cd "$APP_DIR"
git fetch origin
git checkout -B "$BRANCH" "$REF"

if [ ! -f .env ]; then
  echo "오류: $APP_DIR/.env 가 없습니다. .env.example을 참고해 만든 뒤 다시 실행하세요." >&2
  exit 1
fi

if ! command -v pnpm >/dev/null 2>&1; then
  npm i -g pnpm@11 --prefix "$HOME/.local"
fi

pnpm install --frozen-lockfile
pnpm build

# 바람 배치용 Python 가상환경
if [ ! -x wind/.venv/bin/python ]; then
  python3 -m venv wind/.venv
fi
wind/.venv/bin/pip install -q -r wind/requirements.txt
mkdir -p data/wind

pm2 startOrReload ecosystem.config.cjs --update-env
pm2 save

# 기동 직후에는 연결이 거부될 수 있어 몇 번 재시도한다
curl -fsS --retry 10 --retry-delay 1 --retry-connrefused http://127.0.0.1:3090/weather/healthz
echo
```

Run: `chmod +x scripts/deploy.sh scripts/remote-deploy.sh`

- [ ] **Step 3: nginx 설정과 설치 스크립트**

`deploy/nginx/weather.conf`:

```nginx
# /etc/nginx/bytebard.cloud.d/weather.conf — https://bytebard.cloud/weather (weather-app)
# 원본은 저장소 deploy/nginx/weather.conf. 서버에서 손으로 고치지 말고 install.sh로 다시 설치한다.
# bytebard.cloud.conf의 443 server 블록이 include 한다.

location = /weather {
    return 301 /weather/;
}

# 경로를 그대로 넘긴다 (앱이 /weather prefix를 처리)
location ^~ /weather/ {
    proxy_pass http://127.0.0.1:3090;
    proxy_http_version 1.1;
    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Connection        "";
    proxy_read_timeout 30s;
}
```

`deploy/nginx/install.sh`:

```bash
#!/usr/bin/env bash
# 서버에서 root로 한 번 실행: sudo bash /home/rocky/weather-app/deploy/nginx/install.sh
# bytebard.cloud.conf 의 443 server 블록에 include 한 줄을 넣고 weather.conf 를 설치한다.
# 다른 conf·인증서는 건드리지 않는다. nginx -t 나 reload 가 실패하면 원래대로 되돌린다.
set -euo pipefail

if [ "$(id -u)" -ne 0 ]; then
  echo "root 권한이 필요합니다: sudo bash $0" >&2
  exit 1
fi

SRC="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)/weather.conf"
SITE_CONF=/etc/nginx/conf.d/bytebard.cloud.conf
INC_DIR=/etc/nginx/bytebard.cloud.d
DEST="$INC_DIR/weather.conf"
INCLUDE="include $INC_DIR/*.conf;"

# 443 블록을 하나로 특정할 수 있을 때만 진행한다
if [ "$(grep -c 'listen 443 ssl;' "$SITE_CONF")" -ne 1 ]; then
  echo "오류: $SITE_CONF 에서 443 server 블록을 하나로 특정할 수 없습니다. 직접 확인하세요." >&2
  exit 1
fi

BACKUP_DIR="$(mktemp -d)"
cp "$SITE_CONF" "$BACKUP_DIR/site.conf"
if [ -f "$DEST" ]; then cp "$DEST" "$BACKUP_DIR/weather.conf"; fi

restore() {
  cp "$BACKUP_DIR/site.conf" "$SITE_CONF"
  if [ -f "$BACKUP_DIR/weather.conf" ]; then cp "$BACKUP_DIR/weather.conf" "$DEST"; else rm -f "$DEST"; fi
  if nginx -t && systemctl reload nginx; then
    echo "이전 설정으로 되돌렸습니다." >&2
  else
    echo "오류: 복구 후에도 nginx 확인에 실패했습니다. 백업: $BACKUP_DIR" >&2
  fi
}

mkdir -p "$INC_DIR"
install -m 644 "$SRC" "$DEST"
if ! grep -qF "$INCLUDE" "$SITE_CONF"; then
  sed -i "/listen 443 ssl;/a\\    $INCLUDE" "$SITE_CONF"
fi
# SELinux 라벨을 맞춘다
if command -v restorecon >/dev/null 2>&1; then
  restorecon -R "$INC_DIR"
fi

if ! nginx -t; then
  echo "nginx -t 실패: 되돌립니다" >&2
  restore
  exit 1
fi
if ! systemctl reload nginx; then
  echo "nginx reload 실패: 되돌립니다" >&2
  restore
  exit 1
fi
rm -rf "$BACKUP_DIR"

# reload 직후에는 잠깐 실패할 수 있어 재시도한다
for _ in $(seq 10); do
  if curl -fsS "https://bytebard.cloud/weather/healthz"; then
    echo
    echo "완료: https://bytebard.cloud/weather/"
    exit 0
  fi
  sleep 1
done
echo "오류: https://bytebard.cloud/weather/healthz 확인 실패. pm2 weather 상태를 확인하세요." >&2
exit 1
```

- [ ] **Step 4: README에 운영 방법 추가**

`README.md` 끝에 추가:

````markdown
## 개발

```bash
pnpm install
python3 -m venv wind/.venv && wind/.venv/bin/pip install -r wind/requirements-dev.txt
WIND_DATA_DIR=data/wind wind/.venv/bin/python wind/wind_job.py   # 바람 데이터 한 번 받기
pnpm dev                                                         # http://localhost:5173/weather/
pnpm test && (cd wind && .venv/bin/python -m pytest -q)
```

`.env`에 `KMA_SERVICE_KEY`(공공데이터포털 일반 인증키)를 넣는다.

## 배포 (bytebard.cloud)

- 서버 경로 `/home/rocky/weather-app`, pm2 앱 `weather`(3090), `weather-wind`(매시 15분 GFS 갱신)
- 배포: `bash scripts/deploy.sh` (서버에서 build → pm2 reload → healthz 확인)
- 서버 `.env`는 직접 둔다: `scp -P 2022 .env rocky@bytebard.cloud:/home/rocky/weather-app/.env`
- nginx(최초 1회, root): `sudo bash /home/rocky/weather-app/deploy/nginx/install.sh`
- 로그: `pm2 logs weather`, `pm2 logs weather-wind`
````

- [ ] **Step 5: 커밋과 푸시**

```bash
git add ecosystem.config.cjs scripts deploy README.md
git commit -m "chore: pm2·nginx 배포 스크립트" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
git push origin master
```

- [ ] **Step 6: 서버에 .env 두고 첫 배포**

Run:

```bash
ssh -p 2022 rocky@bytebard.cloud 'mkdir -p /home/rocky/weather-app'
scp -P 2022 .env rocky@bytebard.cloud:/home/rocky/weather-app/.env
ssh -p 2022 rocky@bytebard.cloud 'chmod 600 /home/rocky/weather-app/.env'
bash scripts/deploy.sh
```

Expected: 마지막에 `{"ok":true}`. 이어서 확인:

Run: `ssh -p 2022 rocky@bytebard.cloud 'pm2 ls | grep weather; curl -s "http://127.0.0.1:3090/weather/api/forecast?lat=37.5636&lon=126.98" | head -c 200'`
Expected: `weather` online, `weather-wind`는 실행 중이거나 `stopped`(한 번 돌고 끝남). 예보 JSON의 앞부분이 나온다.

Run: `ssh -p 2022 rocky@bytebard.cloud 'sleep 120; pm2 logs weather-wind --lines 5 --nostream; ls /home/rocky/weather-app/data/wind'`
Expected: `사이클 … 저장 (25프레임)`, `latest.json`과 사이클 폴더

- [ ] **Step 7: nginx 적용 (사용자가 직접 실행)**

sudo 비밀번호가 필요하므로 사용자에게 다음을 요청한다:

```
ssh -p 2022 rocky@bytebard.cloud
sudo bash /home/rocky/weather-app/deploy/nginx/install.sh
```

Expected: `완료: https://bytebard.cloud/weather/`

- [ ] **Step 8: 운영 확인**

Run: `curl -sI https://bytebard.cloud/weather | head -3; curl -s https://bytebard.cloud/weather/healthz; curl -s 'https://bytebard.cloud/weather/api/forecast?lat=35.1796&lon=129.0756' | head -c 200`
Expected: `/weather`는 301 → `/weather/`, healthz `{"ok":true}`, 부산 예보 JSON. 다른 서비스(`https://coffee.bytebard.cloud`, `https://youth-worker.bytebard.cloud`)의 상태 코드가 배포 전과 같다.

브라우저로 `https://bytebard.cloud/weather/`를 열어 Task 11·12의 화면 확인 항목을 반복한다.
