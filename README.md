# 🐢 거북이 방지턱 (Turtle Bump)

웹캠으로 앉은 자세를 실시간 분석해 **거북목·허리 굽음**을 감지하고, 알림과 통계로 바른 자세 습관을 만들어 주는 웹 서비스입니다.
브라우저에서 영상이 바로 분석되며(MediaPipe), 측정 기록은 Flask + MariaDB/MySQL 백엔드에 저장됩니다.
다른 웹사이트를 보는 중에도 경고가 표시됩니다.

## 주요 기능

- **실시간 자세 판정**: MediaPipe Pose Landmarker로 어깨·머리 위치를 추적해 cm 단위로 판정
- **초기 기준 자세 설정**: 3초 카운트다운 동안 바른 자세를 측정해 기준값으로 사용
- **자세 상태 분류**: 바른 자세 / 거북목 / 허리 굽음 / 카메라 근접 / 인식 불가
- **경고 알림**: 나쁜 자세가 약 5초 이어지면 경고. 강도는 약함·중간·강함 3단계
  - 약함: Windows 알림
  - 중간: Windows 알림 + 화면 경고창 + 거북이 1마리
  - 강함: Windows 알림 + 화면 경고창 + 거북이 10마리
- **통계 대시보드**: 오늘·최근 7일·최근 30일 기준 측정 시간, 거북목/허리 굽음 횟수, 바른 자세 비율, 평균 지속 시간, 직전 기간과의 비교, 추이 그래프
- **Chrome 확장 프로그램**: Google, Naver 등 다른 사이트에서도 경고창과 거북이 오버레이 표시

## 기술 스택

| 구분 | 사용 기술 |
| --- | --- |
| 프론트엔드 | Vanilla JavaScript (ES Modules), HTML, CSS, Pretendard 폰트 |
| 자세 분석 | MediaPipe Tasks Vision 0.10.14 (Pose Landmarker) |
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
│  └─ js/                       프론트-Flask 연동 브리지 (bcintegration / storage / stats)
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
   │     ├─ b/                  자세 판정 모듈 (neckcare-analyzer.js, turtleneck.js)
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

## 시작하기

### 사전 준비

- Python 3.10 이상
- MariaDB 또는 MySQL 서버
- 웹캠이 있는 PC, Chrome 브라우저
- 인터넷 연결 (MediaPipe 모델·wasm, 폰트를 CDN에서 불러옴)
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
4. 설치 전에 열려 있던 탭은 새로고침

확장 프로그램은 웹앱 페이지(`localhost`/`127.0.0.1`)가 보내는 알림 상태를 받아 모든 탭에 표시합니다.
웹앱 탭을 닫거나 30초 동안 응답이 없으면 알림이 꺼집니다. 자세한 내용은 [`FE/chrome-extension/README.md`](FE/chrome-extension/README.md)를 참고하세요.

## 사용 방법

1. 홈 화면에서 **카메라 연결**을 누르고 권한을 허용합니다.
2. 안내에 따라 3초 동안 바른 자세를 유지해 기준 자세를 설정합니다.
3. 판정이 시작되면 현재 자세가 실시간으로 표시됩니다.
4. 알림 설정에서 강도(약함·중간·강함)를 고르고 미리보기로 확인합니다.
5. 사이드바의 **통계**에서 기간별 기록을 확인합니다.

## 자세 상태

| 상태 | 의미 | 통계 반영 | 경고 |
| --- | --- | --- | --- |
| `GOOD` | 바른 자세 | O | X |
| `TURTLE` | 거북목 (머리가 기준보다 3cm 이상 앞으로) | O | O |
| `WAIST` | 허리 굽음 (어깨가 카메라 쪽으로 6cm 이상 접근) | O | O |
| `CLOSE` | 카메라 근접 (의자를 당겨 앉음) | O | X |
| `NOVIS` | 인식 불가 | X | X |
| `CALIB` | 기준 자세 측정 중 | X | X |
| `IDLE` | 대기 중 | X | X |

판정 임계값과 안정화(EMA + 0.5초 유지) 설정은 `FE/webpage/js/b/turtleneck.js`의 `CONFIG`에 있고, 인터페이스와 예외 처리는 [`FE/webpage/js/b/README.md`](FE/webpage/js/b/README.md)에 정리되어 있습니다.

## API

기본 주소: `http://127.0.0.1:5000/api` (프론트에서 `window.NECKCARE_API_BASE`로 변경 가능)
모든 시각은 UTC epoch 밀리초 정수이며 요청 필드는 snake_case입니다.

| 메서드 | 경로 | 설명 |
| --- | --- | --- |
| `POST` | `/sessions` | 측정 세션 생성 (`started_at_ms`) |
| `POST` | `/sessions/{id}/events` | 자세 이벤트 저장 (요청당 최대 500개) |
| `POST` | `/sessions/{id}/end` | 세션 종료 (미전송 이벤트 함께 전송 가능) |
| `GET` | `/summary/home?as_of_ms=` | 홈 화면 누적 측정 시간 |
| `GET` | `/stats?period=&unit=&as_of_ms=` | 통계. `period`: `today` \| `week` \| `month`, `unit`: `day` \| `week` |

오류 응답은 `{ "error": { "reason": "...", "message": "..." } }` 형식입니다.

### 데이터베이스

| 테이블 | 내용 |
| --- | --- |
| `sessions` | 측정 세션 (시작·종료 시각, 전체·유효 측정 시간) |
| `posture_states` | 자세 상태 코드 정의 (한글 라벨, 통계 반영 여부, 경고 여부) |
| `posture_events` | 상태 구간 이벤트 (UUID, 세션, 상태, 시작·종료 시각, 경고 발생 여부, 확정 여부) |

## 문제 해결

| 증상 | 확인할 것 |
| --- | --- |
| "백엔드 서버에 연결하지 못했습니다" | `BE/app.py`가 실행 중인지, 5000번 포트가 비어 있는지 확인 |
| `python app.py`에서 DB 오류 | `.env` 값, DB 서버 실행 여부, `turtle_bump` 데이터베이스 생성 여부 확인 |
| 카메라 권한 거부 | 주소창의 카메라 권한을 허용하고 `localhost`로 접속했는지 확인 |
| 카메라를 읽을 수 없음 | 화상회의 앱 등 카메라를 쓰는 다른 프로그램 종료 |
| 모델 로딩 실패 | 인터넷 연결 확인 (CDN에서 MediaPipe 모델을 내려받음) |
| 다른 사이트에 경고가 안 뜸 | 확장 프로그램 설치 후 탭 새로고침. `chrome://` 페이지, 새 탭, 웹스토어, 전체화면 영상 중에는 동작하지 않음 |

## 개발 참고

- `FE/chrome-extension/overlay.js`와 `FE/webpage/js/alert-overlay.js`는 **같은 파일**입니다. 한쪽을 수정하면 다른 쪽에도 복사하세요.
- `FE/webpage/js/b/turtleneck.js`는 판정 로직 원본입니다. 수정하면 `b/README.md`의 검증 절차를 다시 확인하세요.
- `BE/.env`는 `.gitignore`에 포함되어 있습니다. DB 비밀번호가 들어 있으니 저장소나 압축 파일로 공유하지 마세요.
- Flask 서버는 `debug=True`, CORS 전체 허용(`*`)으로 개발용 설정입니다. 배포 시에는 변경이 필요합니다.
