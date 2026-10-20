# 🐢 거북이 방지턱 (Turtle Bump)

웹캠으로 앉은 자세를 실시간 분석해 **거북목·허리 굽음**을 감지하고, 알림과 통계로 바른 자세 습관을 만들어 주는 웹 서비스입니다.
영상은 브라우저 안에서만 분석되며(MediaPipe) 밖으로 전송되지 않습니다. 측정 기록(상태 구간)만 Flask + MariaDB/MySQL 백엔드에 저장됩니다.
Chrome 확장 프로그램을 설치하면 다른 웹사이트를 보는 중에도 경고가 표시됩니다.

## 목차

1. [주요 기능](#주요-기능)
2. [기술 스택](#기술-스택)
3. [프로젝트 구조](#프로젝트-구조)
4. [동작 흐름](#동작-흐름)
5. [시작하기](#시작하기)
6. [사용 방법](#사용-방법)
7. [자세 판정](#자세-판정)
8. [판정 모듈 인터페이스](#판정-모듈-인터페이스)
9. [알림과 Chrome 확장 프로그램](#알림과-chrome-확장-프로그램)
10. [백엔드 API와 데이터베이스](#백엔드-api와-데이터베이스)
11. [문제 해결](#문제-해결)
12. [개발 참고](#개발-참고)

---

## 주요 기능

- **실시간 자세 판정**: MediaPipe Pose·Face Landmarker로 머리와 어깨 위치를 추적해 cm 단위로 판정
- **초기 기준 자세 설정**: 3초 카운트다운 동안 바른 자세를 측정해 기준값으로 사용
- **자세 상태 분류**: 바른 자세 / 거북목 / 허리 굽음 / 카메라 근접 / 인식 불가
- **경고 알림**: 나쁜 자세가 약 5초 이어지면 경고, 강도는 약함·중간·세게 3단계
- **통계 대시보드**: 오늘·최근 7일·최근 30일 기준 측정 시간, 거북목/허리 굽음 횟수, 바른 자세 비율, 평균 지속 시간, 직전 기간과의 비교, 추이
- **Chrome 확장 프로그램**: Google, Naver 등 다른 사이트에서도 경고창과 거북이 오버레이 표시

## 기술 스택

| 구분 | 사용 기술 |
| --- | --- |
| 프론트엔드 | Vanilla JavaScript (ES Modules), HTML, CSS, Pretendard 폰트 |
| 자세 분석 | MediaPipe Tasks Vision 0.10.14 (Pose Landmarker, Face Landmarker) |
| 백엔드 | Python, Flask 3, Flask-CORS, python-dotenv |
| 데이터베이스 | MariaDB / MySQL (mysqlclient) |
| 확장 프로그램 | Chrome Extension Manifest V3 |

## 프로젝트 구조

```
turtle_bump
├─ BE/                          백엔드
│  ├─ app.py                    Flask API 서버 (세션·이벤트 저장, 통계 계산, DB 테이블 생성)
│  ├─ requirements.txt
│  ├─ .env                      DB 접속 정보 (Git에 올리지 않음)
│  └─ js/                       Flask 연동 브리지 (bcintegration / storage / stats). 현재 FE에서는 import하지 않음
└─ FE/
   ├─ webpage/                  웹 애플리케이션
   │  ├─ index.html
   │  ├─ server.py              정적 파일 개발 서버 (포트 5173)
   │  ├─ start.bat              Windows 실행 스크립트
   │  ├─ css/styles.css
   │  └─ js/
   │     ├─ main.js             앱 진입점, 전체 연결
   │     ├─ pose.js             카메라 연결
   │     ├─ calibration.js      초기 자세 설정 진행
   │     ├─ posture-adapter.js  판정 결과 → 화면·알림 상태 변환, 5초 경고 정책
   │     ├─ session-recorder.js 측정 세션·자세 이벤트 구간 생성 및 전송
   │     ├─ backend-api.js      Flask API 클라이언트
   │     ├─ stats-source.js     통계 데이터 공급
   │     ├─ notify.js           알림 강도 설정과 알림 실행
   │     ├─ alert-overlay.js    경고창 + 거북이 오버레이
   │     ├─ state.js            상태 저장소
   │     ├─ b/                  자세 판정 모듈
   │     │  ├─ neckcare-analyzer.js   판정 모듈 (createNeckCareAnalyzer)
   │     │  └─ turtleneck.js          판정 로직 (Python v4의 JS 이식판)
   │     ├─ components/         Sidebar, Header, CameraPreview, PostureStatus,
   │     │                      PostureTips, NotifySettings, StatsPage
   │     ├─ ui/toast.js
   │     └─ utils/format.js
   └─ chrome-extension/         Chrome 확장 프로그램
      ├─ manifest.json
      ├─ background.js          알림 상태 저장, 웹앱 탭 종료 시 해제
      ├─ content.js             모든 페이지에 주입, 알림 표시
      ├─ overlay.js             경고창 + 거북이 (webpage/js/alert-overlay.js와 같은 파일)
      └─ popup.html / popup.js  상태 표시, 현재 탭 미리보기
```

## 동작 흐름

```
웹캠 → MediaPipe(브라우저) → 자세 판정 모듈(b/)
                                  │
                                  ▼
                        posture-adapter.js ──► 화면 상태 · 알림(notify.js)
                                  │                      │
                                  ▼                      ▼ postMessage
                        session-recorder.js        Chrome 확장 프로그램
                                  │                (다른 사이트에 경고 표시)
                                  ▼ REST API
                        Flask (BE/app.py) ──► MariaDB/MySQL
                                  │
                                  ▼
                            통계 대시보드
```

---

## 시작하기

### 사전 준비

- Python 3.9 이상 (`requirements.txt`의 Flask 3.x 기준)
- MariaDB 또는 MySQL 서버
- 웹캠이 있는 PC, Chrome 브라우저
- 인터넷 연결 (MediaPipe 모델·wasm, 폰트를 CDN에서 불러옴. 처음 로딩에는 시간이 걸리고 이후 캐시됨)
- `mysqlclient` 빌드 환경 (Windows는 보통 미리 빌드된 휠로 설치됨)

### 1. 데이터베이스 준비

`BE/app.py`는 **테이블은 자동 생성하지만 데이터베이스 자체는 만들지 않습니다.** 먼저 DB를 생성하세요.

```sql
CREATE DATABASE turtle_bump CHARACTER SET utf8mb4;
```

### 2. 백엔드 실행

```powershell
cd BE
python -m venv venv
venv\Scripts\activate
pip install -r requirements.txt
```

`BE/.env` 파일을 만들고 본인 환경에 맞게 작성합니다.

```env
DB_HOST=127.0.0.1
DB_PORT=3306
DB_USER=root
DB_PASSWORD=비밀번호
DB_NAME=turtle_bump
```

```powershell
python app.py
```

실행하면 테이블(`sessions`, `posture_states`, `posture_events`)을 확인·생성한 뒤 `http://127.0.0.1:5000`에서 API 서버가 시작됩니다.

### 3. 프론트엔드 실행

```powershell
cd FE\webpage
start.bat
```

`start.bat`이 개발 서버를 켜고 브라우저를 엽니다. 직접 실행하려면 `python server.py`를 실행한 뒤 `http://localhost:5173`으로 접속하세요.
카메라 권한은 `localhost`에서만 허용되므로 반드시 `localhost` 또는 `127.0.0.1`로 접속해야 합니다.

### 4. Chrome 확장 프로그램 설치 (선택)

1. Chrome 주소창에 `chrome://extensions` 입력
2. 오른쪽 위 **개발자 모드** 켜기
3. **압축해제된 확장 프로그램을 로드합니다** → `FE/chrome-extension` 폴더 선택
4. 설치 전에 열려 있던 탭(웹앱 포함)은 새로고침

## 사용 방법

1. 홈 화면에서 **카메라 연결**을 누르고 권한을 허용합니다. 연결되면 배지가 "카메라 ON (클릭하여 끄기)"로 바뀌고, 이 배지를 누르면 카메라가 꺼지면서 측정 세션이 종료됩니다.
2. 안내에 따라 3초 동안 바른 자세를 유지해 기준 자세를 설정합니다.
3. "거북목 판정 시작!"이 뜨면 현재 자세가 실시간으로 표시됩니다.
4. 알림 설정에서 강도(약함·중간·세게)를 고르고 미리보기로 확인합니다.
5. 사이드바의 **통계**에서 기간(오늘·7일·30일)과 보기 단위(일별·주별)를 고르고 **분석 결과 불러오기**를 누르면 기록을 조회합니다. 측정 중이면 그 시점까지의 기록이 함께 반영됩니다. 통계 화면으로 이동해도 카메라와 판정은 유지됩니다.

---

## 자세 판정

판정은 `FE/webpage/js/b/turtleneck.js`가 하고, `neckcare-analyzer.js`가 카메라 영상 입력·기준 자세 측정·결과 전달을 감쌉니다.
`turtleneck.js`는 Python v4 판정 로직의 JS 이식판이며, 같은 랜드마크를 넣었을 때 Python과 판정이 같은지 프레임 단위로 비교했습니다(33개 case, 9,208프레임, 상태 불일치 0개).
이 비교는 판정 로직만 확인한 것이고, Python(구버전 Pose)과 JS(PoseLandmarker) 사이의 랜드마크 값 차이는 측정하지 않았습니다.

### 상태

판정 순서는 허리 굽음 → 거북목 → 카메라 근접 → 바른 자세입니다. 상태는 하나만 나옵니다.

| 상태 | 의미 | 전환 조건 | 통계 반영 | 경고 |
| --- | --- | --- | --- | --- |
| `GOOD` | 바른 자세 | 아래에 모두 해당하지 않음 | O | X |
| `TURTLE` | 거북목 | 머리가 기준보다 3.0cm 이상 앞으로 (`FWD_TH`) | O | O |
| `WAIST` | 허리 굽음 | 어깨가 6.0cm 이상 접근하고 머리가 2.0cm 이상 앞으로 또는 어깨가 3.0cm 이상 내려감. 어깨가 5.0cm 이상 내려가면 접근과 무관하게 해당 | O | O |
| `CLOSE` | 화면에 너무 가까움 (의자를 당김) | 어깨가 10.0cm 이상 접근하고 머리 앞쪽 이동은 2.0cm 미만 | O | X (안내만) |
| `NOVIS` | 인식 불가 | 얼굴·Pose 없음, 어깨 visibility 0.5 미만, 어깨폭 1px 미만 | X | X |
| `CALIB` | 기준 자세 측정 중 | `CALIB_SEC` 3.0초 동안 | X | X |
| `IDLE` | 대기 중 | 측정 전, 측정 실패, `stop()` 이후 | X | X |

**화면별 표기**: 같은 상태가 화면에 따라 다르게 표시됩니다.

| 상태 | 홈 화면·알림 | 통계 화면 (응답 `type`) |
| --- | --- | --- |
| `GOOD` | 바른 자세 | 바른 자세 (`good`) |
| `TURTLE` | 거북목 주의 → 거북목 자세 | 거북목 (`turtle`) |
| `WAIST` | 허리 굽음 주의 → 허리 굽음 자세 | 새우등 (`shrimp`) |
| `CLOSE` | 바른 자세 | 기타 (`other`, 분포에만 포함) |
| `NOVIS` | 자세 확인 중 | 측정 시간에서 제외 |

임계값은 `turtleneck.js`의 `CONFIG`에서 조정합니다(예: `PostureV4.CONFIG.FWD_TH = 3.5`).

**안정화**: 판정값에 EMA(`EMA_ALPHA` 0.3, 프레임 단위)를 적용하고, 같은 판정이 `HOLD_SEC` 0.5초 이상 이어져야 상태를 바꿉니다. `GOOD`/`TURTLE`/`WAIST`/`CLOSE`는 이 안정화를 거친 값이고, `NOVIS`는 안정화 없이 그 프레임 값을 바로 냅니다. `NOVIS`가 끝나면 직전 안정 판정과 EMA를 이어서 씁니다.

**상태와 함께 붙는 플래그**

| 플래그 | 의미 |
| --- | --- |
| `gated` | 고개를 25° 이상 돌려서 판정을 보류하고 직전 판정을 유지하는 중 |
| `reanchoring` | 의자 이동을 감지해 기준 갱신을 기다리는 중 (1.8초) |
| `reference_updated` | 이번 프레임에 기준이 새 위치로 갱신됨 (한 번만 true) |
| `needs_recalibration` | "다시 설정 필요" 확정 (아래 참고) |

### 초기 기준 자세 설정

| 항목 | 내용 |
| --- | --- |
| 시작 | `beginBaseline(video)`가 끝나는 순간 3초 수집 시작. 모델 로딩·video 준비·준비 추론은 그 전에 끝냄 |
| 수집 방법 | `video`에서 직접 프레임을 읽음 (`requestVideoFrameCallback` + watchdog 타이머). 같은 프레임은 다시 분석하지 않음 |
| 성공 조건 | 3초 동안 유효 프레임(얼굴 행렬 + 양 어깨 visibility 0.5 이상)이 10개 이상 |
| 기준값 | 유효 프레임들의 중앙값 (`k`, `sh`, `sh_y`, `z_head`, `yaw`, `pitch`, `f`, `offset`, `neck`) |
| 저장 위치 | 메모리에만 있음. 브라우저 저장소나 서버에 저장하지 않음 |
| 성공 반환 | `{ ok: true, base, frames, valid_frames, delegate }` |
| 실패 반환 | `{ ok: false, reason, frames, valid_frames, missing }` (`missing`은 미검출 사유별 프레임 수) |
| 실패 사유 | `not_visible`(유효 프레임 부족), `timeout`, `input_ended`, `input_stalled`, `cancelled`(진행 중 stop), `not_started`(begin 없이 호출) |
| 초기화 | `beginBaseline`과 `stop`이 매번 새 `PostureTracker`를 만들어 이전 기준값·EMA가 남지 않음 |
| 의자 위치 변경 | 재측정 없이 자동으로 기준 갱신 (어깨 거리 8cm 이상 변화 후 1.8초 안정) |
| 설정 중 탭·화면 이탈 | 탭을 숨기거나 홈이 아닌 화면으로 이동하면 설정을 취소하고, 돌아오면 카메라가 연결되어 있을 때 처음부터 다시 시작 (`calibration.js`). 판정이 시작된 뒤에는 카메라와 판정을 그대로 유지 |

수집하는 3초 동안의 프레임 수는 fps에 따라 다릅니다(30fps면 약 90개).

### 경고 정책 (`posture-adapter.js`)

판정 결과를 화면·알림 상태로 바꾸는 규칙입니다. 판정 로직은 건드리지 않고 결과만 읽습니다.

| 판정 | 화면 상태 | 타이머 | 경고 |
| --- | --- | --- | --- |
| `GOOD`, `CLOSE` | `good` (CLOSE도 별도 문구 없이 바른 자세로 표시) | 초기화 | 끔 |
| `TURTLE`, `WAIST` | `warn`("거북목 주의" / "허리 굽음 주의"). 5초 이어지면 `bad`("거북목 자세" / "허리 굽음 자세") | 누적 | 5초 이어지면 시작 |
| `NOVIS` | `unknown` ("자세 확인 중") | 일시정지 (지난 시간 유지, 시간은 더하지 않음) | 진행 중인 경고는 유지 |
| `needs_recalibration` | `unknown` ("자세 판단 어려움") | 초기화 | 끄고 "자세 판단이 어렵습니다! 초기자세 설정을 다시 해주세요" 경고창만 표시(거북이 없음). 초기 자세 설정을 다시 시작하기 전까지 이후 결과는 화면에 반영하지 않음 |
| 결과 없음 (시작 전, `IDLE`/`CALIB`) | `unknown` | 초기화 | 끔 |

- 경고 기준은 `BAD_ALERT_MS` = 5000ms입니다. B의 안정화(0.5초)를 포함해 나쁜 자세가 시작된 시점부터 셉니다.
- 거북목 ↔ 허리 굽음으로 바뀌어도 하나의 나쁜 자세 구간으로 보고 이어서 셉니다. 경고 문구만 바뀝니다.
- 경고 시작 시점은 서버에 저장하는 `alert_triggered` 값이 됩니다. 거북목에서 허리 굽음으로 이어진 경고는 새 경고로 치지 않습니다.
- 입력이 끊기면(`input_stalled`, `input_ended`) 타이머를 일시정지하고, 진행 중인 경고는 유지합니다.
- 판정 모듈에는 목 각도 값이 없어서 `posture.angle`은 `null`입니다.

### 인식 불가와 예외

| 상황 | 동작 |
| --- | --- |
| 얼굴이 잠깐 안 잡힘 | `NOVIS` (`reason: no_face` / `no_face_matrix`). 직전 판정과 EMA는 유지하고 다시 보이면 이어서 판정 |
| 어깨 좌표 누락 | `NOVIS` (`reason: shoulders_not_visible` / `no_pose` / `shoulders_invalid`) |
| 렌즈 가림 | 자리 비움과 구분할 수 없음 (`no_face`) |
| 카메라 입력 중단 | 트랙 종료는 `input_ended`, 2초간 새 프레임이 없으면 `input_stalled` 오류. 그동안 결과는 보내지 않음 |
| 모델 로딩 실패 | GPU → CPU로 한 번 재시도. 둘 다 실패하면 `model_load_failed`로 `beginBaseline`이 예외를 던짐 |
| 추론 예외 | `inference_error`를 알리고 그 프레임만 건너뜀 |

**잠깐 미검출 vs 다시 설정 필요**: 판정 코드 자체에는 둘을 구분하는 조건이 없어서 `NOVIS`가 몇 초든 이어질 수 있습니다. 그래서 모듈에 `recalibrateAfterNovisMs` 옵션을 두었고, `main.js`에서 **5분**으로 설정되어 있습니다. `NOVIS`가 5분 이어지면 `needs_recalibration: true`가 되어 "초기자세 설정을 다시 해주세요" 안내가 표시됩니다.

---

## 판정 모듈 인터페이스

`neckcare-analyzer.js`는 `createNeckCareAnalyzer(options)` 하나를 export하는 ES module입니다. `main.js`가 `import('./b/neckcare-analyzer.js')`로 불러와 `posture-adapter.js`로 감싼 뒤 `calibration.registerAnalyzer()`로 등록합니다. 브라우저 콘솔에서는 `window.neckcare.registerAnalyzer(analyzer)`로도 등록할 수 있습니다.

```js
import { createNeckCareAnalyzer } from './b/neckcare-analyzer.js';

const analyzer = createNeckCareAnalyzer({
  onResult(r) { /* 판정 결과 */ },
  onError(e)  { /* 오류 */ },
});
window.neckcare.registerAnalyzer(analyzer);   // 카메라 연결 전에 등록
```

### 옵션

| 옵션 | 기본값 | 설명 |
| --- | --- | --- |
| `onResult` | `null` | 결과 콜백 |
| `onError` | `null` | 오류 콜백. 없으면 `console.warn` |
| `emit` | `'change'` | `'change'`면 상태나 플래그가 바뀔 때만 결과를 보냄. `'frame'`이면 분석한 프레임마다 보냄 |
| `delegate` | `'GPU'` | `'CPU'`로 고정 가능 |
| `cpuFallback` | `true` | GPU 모델 생성에 실패하면 CPU로 한 번 더 시도 |
| `wasmBase`, `poseModel`, `faceModel` | jsdelivr / googleapis URL | 로컬 경로로 바꿀 수 있음 |
| `recalibrateAfterNovisMs` | `null` | `NOVIS`가 이 시간 이상 이어지면 `needs_recalibration: true`. `main.js`는 5분으로 지정 |
| `inputStallMs` | `2000` | 새 영상 프레임이 이 시간 동안 없으면 `input_stalled` 오류 |
| `videoReadyTimeoutMs` | `5000` | `beginBaseline`에서 video가 재생 가능해질 때까지 기다리는 최대 시간 |
| `baselineGraceMs` | `2000` | `finishBaseline`이 3초 수집 완료를 추가로 기다리는 여유 시간 |
| `watchdogMs` | `200` | `requestVideoFrameCallback`이 이 시간 동안 오지 않으면 타이머로 대신 분석 (video가 숨겨졌을 때 등) |

### 메서드

| 메서드 | 반환 | 하는 일 |
| --- | --- | --- |
| `beginBaseline(video)` | `Promise<void>` | 이전 진행과 기준값을 모두 버림 → 모델 로딩(처음 한 번만) → video 준비 대기 → 준비 추론 1회 → 3초 수집 시작 → 추론 루프 시작 |
| `finishBaseline()` | `Promise<{ok, …}>` | 3초 수집이 끝날 때까지 기다린 뒤 결과 반환. 던지지 않음 |
| `start(cb?)` | `undefined` | 결과 전달 시작. 현재 결과를 바로 한 번 보냄. 기준 자세가 없으면 경고만 하고 무시 |
| `stop()` | `undefined` | 루프·타이머·수집을 정리하고 기준값 삭제. 대기 중인 `finishBaseline`은 `cancelled`로 끝남. 모델은 재사용하려고 남겨 둠 |
| `onResult(cb)` / `onError(cb)` | 해제 함수 | 콜백 추가 |
| `getLatest()` | 결과 또는 `null` | 마지막으로 계산한 결과 |
| `getDebug()` | 객체 | `phase`, `fps`, `delegate`, `calib`(수집 프레임 수와 미검출 사유), `lines`(수치), `config` |
| `dispose()` | `undefined` | `stop()` + 모델 해제 |

**호출 순서**: `beginBaseline` → `finishBaseline` → (`ok`이면) `start`, (실패면) `stop`. 언제든 `stop()`을 부르면 처음 상태로 돌아가고, `start()` 전에는 결과를 보내지 않습니다. 늦게 끝난 비동기 작업은 내부 진행 번호로 걸러서 무시합니다.

**`beginBaseline`이 예외를 던지는 경우**

| 상황 | 예외 | `onError` type |
| --- | --- | --- |
| video 인자가 아님 | `TypeError` | — |
| 모델 로딩 실패 (GPU와 CPU 모두) | 원래 오류 | `model_load_failed` |
| 5초 안에 video가 준비되지 않음 | `video_not_ready` | `video_not_ready` |
| 준비 추론 실패 | 원래 오류 | `inference_error` |
| 진행 중에 `stop()`이나 새 `beginBaseline`이 호출됨 | `cancelled` | — |

### 오류 객체 (`onError`)

```js
{ type, message, error, detail, timestamp /* Date.now() ms */ }
```

| type | 언제 | 결과 전달 |
| --- | --- | --- |
| `model_load_failed` | 모델/wasm 로딩 실패 | `beginBaseline`이 예외를 던짐 |
| `video_not_ready` | video 준비 시간 초과 | `beginBaseline`이 예외를 던짐 |
| `inference_error` | `detectForVideo` 예외. 연속으로 나면 처음 한 번만 알림 | 그 프레임은 건너뜀 |
| `input_ended` | video의 MediaStreamTrack이 `ended`가 됨 | 결과 전달이 멈춤 |
| `input_stalled` | `inputStallMs` 동안 새 프레임이 없음 | 결과 전달이 멈춤. 프레임이 다시 오면 자동으로 이어짐 |

### 판정 결과 객체 (`onResult`)

```ts
{
  source: 'neckcare-b',
  state: 'GOOD'|'TURTLE'|'WAIST'|'CLOSE'|'NOVIS',
  prev_state: string|null,      // 직전의 '다른' state
  stabilized: boolean,          // GOOD/TURTLE/WAIST/CLOSE = true, NOVIS = false (프레임 원본)
  alert: boolean,               // TURTLE || WAIST
  state_since: number,          // 이 state가 시작된 시각, Date.now() ms
  duration_ms: number,          // 이 state가 이어진 시간
  reason: null|'no_face'|'no_face_matrix'|'no_pose'|'shoulders_not_visible'|'shoulders_invalid', // NOVIS일 때만
  gated: boolean,
  reanchoring: boolean,
  reference_updated: boolean,
  novis_ms: number,             // NOVIS가 이어진 시간 (아니면 0)
  needs_recalibration: boolean,
  timestamp: number,            // Date.now() ms
  perf_time: number,            // performance.now() ms
  values: {                     // EMA 적용값 (NOVIS 동안에는 마지막 값)
    fwd, appr, drop, dist,      // cm (판정에 사용)
    yaw, pitch,                 // 도, 기준 대비
    neck, z_head, z_sh, Y_sh    // cm (표시용)
  } | null
}
```

`alert`는 판정 모듈 기준 정의(`TURTLE` 또는 `WAIST`)이고 5초 경고 정책과는 별개입니다. 판정에는 신뢰도 값이 없어서 보내지 않습니다(어깨 visibility 0.5 기준만 사용).

### 실행 환경

| 항목 | 내용 |
| --- | --- |
| MediaPipe | `@mediapipe/tasks-vision@0.10.14`. 웹앱과 같은 jsdelivr URL을 import하므로 모듈이 한 번만 로드됨 |
| Pose | `PoseLandmarker`, `runningMode: 'VIDEO'`, `numPoses: 1`, confidence 0.5, 모델 `pose_landmarker_full` float16 |
| Face | `FaceLandmarker`, `runningMode: 'VIDEO'`, `numFaces: 1`, confidence 0.5, `outputFacialTransformationMatrixes: true`(필수), 모델 `face_landmarker` float16 |
| 대리자 | GPU(WebGL), 실패하면 CPU. 실제 값은 `getDebug().delegate` |
| 추론 호출 | `detectForVideo(video, ts)`. Pose와 Face에 같은 `ts`(단조 증가 정수 ms) |
| 추론 반복 | `requestVideoFrameCallback`(새 프레임마다) + watchdog 타이머. `requestAnimationFrame`은 쓰지 않음 |
| 입력 형식 | `HTMLVideoElement`를 그대로 넣음. 해상도 제약 없음 (세로 화각 63° 가정으로 초점거리 계산) |
| 좌우 반전 | 반전하지 않은 원본을 씀. 판정값이 모두 거리·높이·초기값 대비 각도라 반전과 무관 |
| 권한 | 카메라 권한은 웹앱이 받음. 판정 모듈은 카메라를 열지 않음 |
| 네트워크 | 처음 로딩할 때 jsdelivr와 storage.googleapis.com에 접근. 영상은 밖으로 보내지 않음 |

---

## 알림과 Chrome 확장 프로그램

### 알림 강도

| 강도 | 표시 |
| --- | --- |
| 약함 | Windows 알림 |
| 중간 | Windows 알림 + 화면 경고창 + 거북이 1마리 |
| 세게 | Windows 알림 + 화면 경고창 + 거북이 10마리 |

강도는 브라우저에 저장되어 새로고침해도 유지되며 기본값은 "중간"입니다. (코드 내부 값은 `weak` / `medium` / `strong`이고, 화면에는 "세게"로 표시됩니다.)

경고 문구는 자세 종류별로 다릅니다("거북목 자세가 감지되었어요" / "허리 굽음 자세가 감지되었어요"). 알림 설정 화면의 **미리보기**는 약 3초 동안만 표시되며 실제 알림 상태나 통계를 바꾸지 않습니다.

### 확장 프로그램 동작

웹앱에서 실제 거북목·허리 굽음 알림이 켜지면, 다른 웹사이트에도 화면 중앙 상단 경고창과 거북이를 표시합니다. 정상 자세로 돌아오면 사라집니다.

1. 웹앱 페이지(`localhost` / `127.0.0.1`이면서 `<meta name="neckcare-app">`가 있는 페이지)가 `window.postMessage`로 알림 상태·강도를 보냅니다.
2. `content.js`가 이를 `background.js`로 넘기고, `background.js`가 `chrome.storage`에 저장합니다.
3. 모든 탭의 `content.js`가 저장된 상태를 보고 경고창을 표시하거나 제거합니다.

- 웹앱 탭은 10초마다 신호(heartbeat)를 보내고, 30초 동안 신호가 없으면 알림이 꺼진 것으로 봅니다. 웹앱 탭을 닫거나 새로고침해도 꺼집니다.
- 팝업의 **현재 탭에서 미리보기**는 지금 보고 있는 탭에만 약 3초 표시하며 실제 알림 상태는 바꾸지 않습니다.

| 파일 | 역할 |
| --- | --- |
| `overlay.js` | 경고창 + 거북이 오버레이. `webpage/js/alert-overlay.js`와 같은 파일 |
| `content.js` | 모든 페이지에 주입. 알림 표시, 웹앱 → background 중계, 팝업 미리보기 |
| `background.js` | 알림 상태 저장, 웹앱 탭이 닫히면 해제 |
| `popup.html`, `popup.js` | 상태·강도 표시, 현재 탭 미리보기 |

**작동하지 않는 곳**: `chrome://` 페이지, 새 탭 페이지, Chrome 웹스토어, 다른 확장 프로그램 페이지, 내장 PDF 뷰어, `file://`(확장 프로그램 설정에서 "파일 URL에 대한 액세스 허용" 필요), 시크릿 창(별도 허용 필요), 전체화면 영상 재생 중.

---

## 백엔드 API와 데이터베이스

기본 주소: `http://127.0.0.1:5000/api` (프론트에서 `window.NECKCARE_API_BASE`로 변경 가능)
모든 시각은 UTC epoch 밀리초 정수이며 요청 필드는 snake_case입니다.

| 메서드 | 경로 | 설명 |
| --- | --- | --- |
| `POST` | `/sessions` | 측정 세션 생성 (`started_at_ms`) |
| `POST` | `/sessions/{id}/events` | 자세 이벤트 저장 (요청당 최대 500개) |
| `POST` | `/sessions/{id}/end` | 세션 종료 (미전송 이벤트 함께 전송 가능) |
| `GET` | `/summary/home?as_of_ms=` | 홈 화면 누적 측정 시간 (`as_of_ms` 생략 시 서버 현재 시각) |
| `GET` | `/stats?period=&unit=&as_of_ms=` | 통계. `period`: `today` \| `week` \| `month`, `unit`: `day` \| `week` |

오류 응답은 `{ "error": { "reason": "...", "message": "..." } }` 형식입니다.

### 세션 기록 규칙 (`session-recorder.js`)

- 카메라 연결에 성공한 시각을 기준으로 세션을 1개 만들고, 첫 구간은 `CALIB`(유효 시간 제외)입니다.
- 상태가 바뀌면 같은 시각 하나로 이전 구간 종료(`is_final: true`)와 새 구간 시작을 맞춥니다. 0초 구간은 보내지 않습니다.
- 전송 시점은 통계 갱신 직전(진행 중 구간은 현재 시각까지의 스냅샷, `is_final: false`), 카메라 OFF, 페이지 종료(`pagehide`)입니다. 페이지 종료 때는 `keepalive` fetch와 `sendBeacon`을 함께 시도하며(성공 여부는 보장되지 않는 최선 노력 전송), 본문이 64KB 미만이 되도록 오래된 이벤트부터 뺍니다.
- 전송에 실패한 구간은 같은 UUID·종료 시각으로 보관했다가 재시도합니다(2·5·10·30·60초 간격).
- 카메라를 다시 연결하면 새 세션으로 따로 진행합니다.

### 통계 집계

- 측정 시간은 `GOOD`, `TURTLE`, `WAIST`, `CLOSE` 구간의 합입니다. `NOVIS`, `CALIB`, `IDLE`과 종료 시각이 없는(미확정) 구간은 제외됩니다. 조회 기간 경계에 걸친 구간은 기간 안쪽만 잘라서 셉니다.
- 기간은 한국 시간(KST) 기준이고 `as_of_ms`(조회 시각)까지 집계합니다.

| `period` | 현재 구간 | 비교 대상(`prev`) |
| --- | --- | --- |
| `today` | 오늘 0시 ~ 조회 시각 | 어제 0시 ~ 어제 같은 시각 |
| `week` | 오늘 포함 최근 7일 | 그 직전 7일 |
| `month` | 오늘 포함 최근 30일 | 그 직전 30일 |

- `summary`: `measuredSec`(측정 시간), `turtleCount`/`shrimpCount`(해당 기간에 **시작한** 거북목·허리 굽음 구간 수), `goodRatio`(바른 자세 시간 ÷ 측정 시간, %), `avgDurationSec`(거북목·허리 굽음 구간의 평균 지속 시간). 각 항목은 `{ value, prev }` 형태입니다.
- `distribution`: `good` / `turtle` / `shrimp` / `other`(`CLOSE`)별 시간과 비율
- `trend`: 거북목·허리 굽음(`shrimp`) 횟수. `unit`이 `day`면 하루, `week`면 7일 단위 구간이며 `today`는 구간이 하나입니다.
- `records`: `GOOD`/`TURTLE`/`WAIST` 구간 목록 (`id`, `startedAt`, `type`, `durationSec`)
- 기록이 없으면 `summary`·`distribution`은 `null`, `trend`와 `records`는 빈 배열입니다. 홈의 `measured_sec`도 기록이 없으면 `null`이고 화면에는 `--:--:--`로 표시됩니다.
- 조회에 실패하면 통계 화면은 "불러오지 못했어요. 이전 결과를 유지합니다"를 표시하고 이전 정상 데이터를 남깁니다.

### 데이터베이스

| 테이블 | 내용 |
| --- | --- |
| `sessions` | 측정 세션 (시작·종료 시각, 전체·유효 측정 시간) |
| `posture_states` | 자세 상태 코드 정의 (한글 라벨, 통계 반영 여부, 경고 여부) |
| `posture_events` | 상태 구간 이벤트 (UUID, 세션, 상태, 시작·종료 시각, 경고 발생 여부, 확정 여부) |

---

## 문제 해결

| 증상 | 확인할 것 |
| --- | --- |
| "백엔드 서버에 연결하지 못했습니다" | `BE/app.py`가 실행 중인지, 5000번 포트가 비어 있는지 확인 |
| `python app.py`에서 DB 오류 | `.env` 값, DB 서버 실행 여부, `turtle_bump` 데이터베이스 생성 여부 확인 |
| 카메라 권한 거부 | 주소창의 카메라 권한을 허용하고 `localhost`로 접속했는지 확인 |
| 카메라를 읽을 수 없음 | 화상회의 앱 등 카메라를 쓰는 다른 프로그램 종료 |
| 모델 로딩 실패 | 인터넷 연결 확인 (CDN에서 MediaPipe 모델을 내려받음) |
| 기준 자세 설정 실패 | 얼굴과 양 어깨가 화면에 보이게 앉았는지 확인. 원인은 콘솔에서 `B.getDebug().calib`의 `missing`으로 확인 |
| 항상 인식 불가(`NOVIS`)로 나옴 | 어깨가 화면 안에 들어오는지, 조명이 충분한지 확인 |
| 다른 사이트에 경고가 안 뜸 | 확장 프로그램 설치 후 탭 새로고침. 위의 "작동하지 않는 곳"에 해당하는지 확인 |

## 개발 참고

- `FE/chrome-extension/overlay.js`와 `FE/webpage/js/alert-overlay.js`는 **같은 파일**입니다. 한쪽을 수정하면 다른 쪽에도 복사하세요.
- `FE/webpage/js/b/turtleneck.js`는 판정 로직 원본입니다. 임계값이나 판정 순서를 바꾸면 아래 테스트 절차로 다시 확인하세요.
- `BE/.env`는 `.gitignore`에 포함되어 있습니다. DB 비밀번호가 들어 있으니 저장소나 압축 파일로 공유하지 마세요.
- Flask 서버는 `debug=True`, CORS 전체 허용(`*`)으로 개발용 설정입니다. 배포 시에는 변경이 필요합니다.

### 판정 동작 테스트 (브라우저 콘솔)

카메라를 연결하기 **전에** 개발자 도구 콘솔에서 판정 로그를 붙입니다.

```js
const { createNeckCareAnalyzer } = await import('/js/b/neckcare-analyzer.js');
window.B = createNeckCareAnalyzer({
  onResult: (r) => console.log('[B]', r.state, r.reason ?? '', r.gated ? 'gated' : '',
                                r.reanchoring ? 'reanchoring' : '', Math.round(r.duration_ms), r.values),
  onError:  (e) => console.warn('[B error]', e.type, e.message, e.detail),
});
window.neckcare.registerAnalyzer(window.B);
```

카메라를 연결하고 3초 동안 바른 자세를 유지한 뒤, 아래 동작을 각각 5초 이상 유지하며 로그를 확인합니다.

| 재현 동작 | 기대 결과 |
| --- | --- |
| 바른 자세 | `GOOD` |
| 어깨는 그대로 두고 턱을 앞으로 내밈 (3cm 이상) | `TURTLE` |
| 엉덩이를 축으로 상체를 앞으로 숙임 | `WAIST` |
| 자세는 바르게 유지하고 의자를 10cm 이상 화면 쪽으로 당김 | `CLOSE` (1.8초 뒤 `reference_updated`) |
| 그 위치에서 턱을 내밈 | `TURTLE` |
| 고개를 30° 이상 옆으로 돌림 | 직전 state 유지 + `gated` |
| 화면 밖으로 나가기, 렌즈 가리기 | `NOVIS` + `reason` |

실시간 수치는 `setInterval(() => console.log(B.getDebug().lines.join(' | '), 'fps', B.getDebug().fps), 1000)`으로 볼 수 있습니다. 카메라를 해제하면 `stop()`이 호출되어 `B.getDebug().phase === 'idle'`이고 로그가 더 나오지 않아야 합니다.

### 통합 테스트

`NeckCare_통합_테스트.html`의 체크리스트 24개 항목을 직접 수행했고 모두 통과했습니다. 준비 단계는 MariaDB 실행, `python app.py`, `start.bat`, 확장 프로그램 로드, 개발자 도구(Console·Network) 열기입니다.

| 영역 | 항목 | 통과 |
| --- | --- | --- |
| A. 기본 화면·서버 | 첫 화면 로딩, 홈↔통계 전환(`#home`/`#stats`), 홈 누적 시간(`GET /api/summary/home`) | ✅ |
| B. 카메라·초기 자세 설정 | 웹캠 연결과 세션 생성(`POST /api/sessions` 201), 중복 연결 방지, 설정 성공·실패, 설정 중 화면 이동 시 처음부터 재시작, 권한 거부·웹캠 없음 | ✅ |
| C. 자세 판정·알림 | 바른 자세, 거북목 5초 경고(경고창·거북이·Windows 알림 1회), 거북목→허리 굽음 이어짐(문구만 변경, 알림 재발송 없음), 경고 해제, `CLOSE`(경고 없음), `NOVIS`(타이머 정지 후 재개), 알림 강도 3단계와 새로고침 후 유지 | ✅ |
| D. 확장 프로그램 | 다른 사이트에 경고 표시, 바른 자세와 웹앱 탭 종료 시 해제 | ✅ |
| E. 저장·통계 | 측정 중 통계 조회(`POST /events` 후 `GET /stats`), 카메라 OFF 시 세션 종료, DB 저장 확인, 재연결 시 새 세션, 통계 기간·단위·유형 전환, 백엔드 장애 시 동작, 종료 재시도 | ✅ |

DB 저장 확인(#20)은 아래 SQL로 합니다. 시각은 UTC입니다(한국 시간 = +9시간). 세션에 종료 시각이 있고, 이벤트가 시간순으로 끊김 없이 이어지며 모두 `is_final = 1`이고, 경고가 뜬 구간만 `alert_triggered = 1`이어야 합니다.

```sql
SELECT session_id, started_at, ended_at, total_duration_sec, valid_duration_sec
FROM sessions ORDER BY session_id DESC LIMIT 3;

SELECT state, started_at, ended_at, duration_sec, alert_triggered, is_final
FROM posture_events WHERE session_id = ? ORDER BY started_at;   -- ?에 위 세션 번호
```

### 확인된 것과 확인되지 않은 것

| 항목 | 상태 |
| --- | --- |
| Python v4 ↔ `turtleneck.js` 판정 로직 일치 (반전 포함, 9,208프레임) | 확인함 (불일치 0) |
| 실제 사람·카메라로 `GOOD`/`TURTLE`/`WAIST`/`CLOSE`/`NOVIS` 판정과 기준 자세 설정 | 확인함 (통합 테스트 #6, #7, #10~#15) |
| JS PoseLandmarker의 어깨 `visibility` 값 | 확인함. 기준 자세 설정이 성공하려면 어깨 visibility 0.5 이상인 유효 프레임이 필요한데 #6이 통과함 |
| 알림 강도, 경고 해제, 확장 프로그램 표시·해제 | 확인함 (#11~#13, #16, #17) |
| 세션·이벤트 저장, 통계 조회, 백엔드 장애·종료 재시도 | 확인함 (#18~#24) |
| 웹앱 탭이 백그라운드일 때 분석 빈도(fps)와 장시간(수 분 이상) 유지 시 지연 정도 | 미측정. 판정과 알림이 동작하는 것 자체는 통합 테스트 D로 확인함. |
| 숨긴 video(`display:none`)에서의 분석 지속 | 확인함 (내장 브라우저. 영상 프레임 콜백이 멈추면 200ms watchdog 타이머가 대신 분석) |
| Python과 JS의 랜드마크 값 차이(구버전 Pose ↔ PoseLandmarker) | 확인함 |