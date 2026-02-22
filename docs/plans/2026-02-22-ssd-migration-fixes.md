# 2026-02-22 SSD 이전 및 장애 수정 기록

날짜: 2026-02-22

---

## 1. 프로젝트 SSD 이전 (USB 외장 SSD)

| 항목 | 내용 |
|------|------|
| 이전 경로 | `/Users/d43103/Workspace/projects/openclaw-javis` |
| 현재 경로 | `/Volumes/mac_mini/Workspace/openclaw-javis` |
| 이전 일시 | 2026-02-22 |

프로젝트 디렉토리를 내장 디스크에서 USB 외장 SSD로 이전함. 이후 아래 항목들의 하드코딩된 경로 참조가 무효화되어 연쇄 장애 발생.

---

## 2. Gateway TLS 인증서 재생성

### 문제

- iOS 앱 pairing 시 인증서의 `CA:TRUE` 플래그 문제 발생
- TLS fingerprint 캐시 버그로 인해 재연결 시 인증 실패

### 해결

- SAN (Subject Alternative Name) extension 포함하여 새 인증서 재생성
- iOS Reset Onboarding 플로우에 TLS fingerprint 초기화 로직 추가

### 수정 파일

```
apps/shared/OpenClawKit/Sources/OpenClawKit/GatewayTLSPinning.swift
```

---

## 3. macOS 앱 Gateway 연결 실패

### 문제

- `operator.write` scope 누락으로 macOS 앱의 gateway 연결이 거부됨

### 해결

- `~/.openclaw/devices/paired.json`에 `operator.write` scope 수동 추가

---

## 4. Gateway TTS Proxy 503 오류 (오늘의 주요 이슈)

### 증상

iOS Talk Mode에서 TTS 요청 시 다음 오류 반환:

```
HTTP 503: {"ok":false,"error":"talk.ttsBaseUrl not configured"}
```

### 근본 원인

launchd plist 파일이 이전 경로를 하드코딩하고 있었음.

파일 위치: `~/Library/LaunchAgents/ai.openclaw.gateway.plist`

```xml
<!-- 이전 (잘못된 경로 - SSD 이전 후 존재하지 않음) -->
<string>/Users/d43103/Workspace/projects/openclaw-javis/dist/index.js</string>
```

프로젝트가 외장 SSD로 이전된 후 해당 경로가 존재하지 않아 게이트웨이가 start/crash를 반복하고 있었음. 게이트웨이 프로세스 자체가 정상적으로 기동하지 않았으므로 TTS proxy 설정(`talk.ttsBaseUrl`)을 로드할 수 없는 상태였음.

### 해결 절차

1. `scripts/install-javis-fork.sh` 실행 - 새 빌드 + npm 전역 설치 수행

   ```bash
   cd /Volumes/mac_mini/Workspace/openclaw-javis
   scripts/install-javis-fork.sh
   ```

2. plist의 실행 파일 경로를 전역 npm 설치 경로로 수정

   ```xml
   <!-- 수정 후 (올바른 경로) -->
   <string>/opt/homebrew/lib/node_modules/openclaw/dist/index.js</string>
   ```

3. launchd reload 및 게이트웨이 정상 재시작 확인

   ```bash
   launchctl unload ~/Library/LaunchAgents/ai.openclaw.gateway.plist
   launchctl load ~/Library/LaunchAgents/ai.openclaw.gateway.plist
   ```

4. TTS proxy 동작 확인

   ```bash
   curl -sk https://127.0.0.1:18789/v1/audio/speech
   # HTTP 200 응답 확인
   ```

---

## 5. 현재 시스템 구성

| 항목 | 값 |
|------|-----|
| 게이트웨이 실행 파일 | `/opt/homebrew/lib/node_modules/openclaw/dist/index.js` |
| 게이트웨이 버전 | v2026.2.22 |
| launchd plist | `~/Library/LaunchAgents/ai.openclaw.gateway.plist` |
| config | `~/.openclaw/openclaw.json` |
| TTS base URL (config) | `http://192.168.219.106:8031` |

### iOS TTS 요청 흐름

```
iPhone
  └─► gateway proxy (wss://100.127.27.87:18789)
         └─► TTS server (http://192.168.219.106:8031)
```

---

## 향후 주의사항

- launchd plist에 프로젝트 로컬 경로를 직접 참조하지 말 것. npm 전역 설치 경로(`/opt/homebrew/lib/node_modules/openclaw/`) 기준으로 유지한다.
- 프로젝트 이전 또는 디렉토리 변경 시 `scripts/install-javis-fork.sh`를 반드시 재실행하고 plist 경로도 함께 점검한다.
