<p align="center">
  <img src="docs/icon.png" width="96" height="96" alt="K-Debugger">
</p>

<h1 align="center">K-Debugger</h1>

<p align="center">Android 앱에 붙어 로그·네트워크·상태를 실시간으로 들여다보는 macOS 데스크톱 디버깅 도구.</p>

<p align="center">
  <a href="https://github.com/rex-ingeni/K-Debugger/releases/latest"><b>⬇ macOS 다운로드</b></a> ·
  <a href="https://rex-ingeni.github.io/K-Debugger/">소개 페이지</a> ·
  <a href="https://rex-ingeni.github.io/K-Debugger/android.html">Android 연결 가이드</a>
</p>

---

## 다운로드 & 설치

1. [최신 릴리스](https://github.com/rex-ingeni/K-Debugger/releases/latest)에서 `.dmg`를 받는다.
2. DMG를 열어 **K-Debugger.app**을 Applications로 드래그.
3. 서명 전 단계라 첫 실행은 앱을 **우클릭 → 열기**.

## 자동 업데이트

새 버전이 나오면 앱을 켤 때 **⚙ 설정 버튼에 뱃지**가 뜬다. 설정 → **업데이트**를 누르면 앱이 새 빌드를 직접 받아 교체하고 재시작한다(무결성은 SHA-256로 대조).

---

## Android 연결 가이드

연결은 **adb**로 이뤄진다. 자세한 내용은 [연결 가이드 페이지](https://rex-ingeni.github.io/K-Debugger/android.html) 참고.

### 1. 사전 준비
- **adb 설치** — Android SDK Platform-Tools (예: `brew install android-platform-tools`)
- 기기에서 **USB 디버깅 ON** (설정 → 개발자 옵션; 개발자 옵션은 "빌드 번호" 7회 탭)
- USB 연결 후 "USB 디버깅 허용" 승인, 확인:
  ```
  adb devices
  # XXXXXXXX   device   ← 이렇게 뜨면 정상
  ```

### 2. 기기 감지
K-Debugger를 실행하면 연결된 기기를 자동 감지해 **상단 기기 선택**에 표시한다.

### 3. WebView 디버깅 ✅ *지금 동작*
앱 안 WebView를 Chrome DevTools Protocol(CDP)로 직접 디버깅한다. 별도 SDK 없이 adb만으로 동작.
```kotlin
// 대상 앱의 디버그 빌드에서
if (BuildConfig.DEBUG) {
    WebView.setWebContentsDebuggingEnabled(true)
}
```
해당 WebView 화면을 띄운 뒤, K-Debugger의 **WebView 탭**에서 타깃을 고르면 DevTools가 열린다.

### 4. 로그 · 네트워크 · DB · Prefs (SDK 연동)
앱의 로그·네트워크·DB·Preferences는 device측 **KDebugger SDK** 연동으로 본다. SDK는 **연결 · 이벤트 전송(`sendEvent`) · 명령 처리(`commandHandler`)** 만 제공하고, 각 플러그인은 앱의 후킹 지점에서 이 API로 데이터를 흘려보낸다(Flipper 후킹 지점 미러링).

**연결**
```kotlin
// Application.onCreate() — 디버그 빌드에서만
if (BuildConfig.DEBUG) KDebugger.init(this)   // abstract socket "k-debugger" 대기
```
```
adb forward tcp:8088 localabstract:k-debugger   # 데스크톱에서 소켓 매핑
```
프로토콜: 4바이트 빅엔디안 길이 헤더 + UTF-8 JSON. 릴리스에서 `init`을 안 부르면 no-op.

**플러그인별 후킹 원리**
- **로그** — 로그 Tree/래퍼에서 `sendEvent("log", id, ts, {level,tag,message})`
- **네트워크(REST)** — OkHttp `addInterceptor`로 요청·응답을 가로채 `sendEvent("network", …)`
- **네트워크(WebSocket)** — 리스너 래핑, `onOpen/onClose/onSend/onReceive`에서 프레임을 `sendEvent("network", …)`
- **DB** — `commandHandler`에서 데스크톱이 보낸 SQL을 실행하고 rows 반환
- **Prefs** — SharedPreferences 값을 `sendEvent("prefs", …)` 또는 `commandHandler`로 응답

> 모든 후킹은 디버그 빌드에서만(`if (BuildConfig.DEBUG)`) — 릴리스는 no-op.

---

## 참고
- **Android 전용** (iOS 미지원).
- macOS 앱은 Developer ID 서명 전 단계 → 첫 실행 우클릭 열기.
