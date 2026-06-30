# ENT WebXR 임상 앱 — 작업 핸드오프 (Claude Code용)

> 이 문서는 채팅 세션에서 진행한 내용을 Claude Code에서 이어서 작업하기 위한 컨텍스트입니다.
> 사용자: 대구 ENT 의원 원장. 용도: **병원 내 치료 전용**(상용 배포 아님).
> 기기 타깃: **Galaxy XR(Android XR / Chrome WebXR)** 우선, Vision Pro(Safari WebXR) 호환.

---

## 0. 가장 중요한 교훈 — WebXR 배포는 HTTPS 필수

세션에서 가장 오래 막혔던 지점. **반드시 먼저 숙지할 것.**

- WebXR 몰입 세션(`requestSession`)은 **보안 컨텍스트(https)** 에서만 열린다.
- `file://`, `http://`, **임베드 iframe**에서는 `requestSession`이 권한 단계 이전에
  `NotSupportedError: The specified session configuration is not supported` 로 실패한다.
  (`isSessionSupported`는 `true`를 반환해도 `requestSession`은 실패할 수 있음 — 둘은 별개.)
- **Android XR(Galaxy XR) Chrome은 모든 WebXR API에 "3D 매핑·카메라 추적" 권한을 요구**한다.
  https에서 세션을 요청해야 권한 프롬프트가 뜨고, 허용해야 진입된다.
- iframe 임베드 시 `allow="xr-spatial-tracking"` 필요.

### 배포 방법 (확정)
- GitHub Pages 레포: **`silverylaker-cmyk/VRT1`**
- 루트에 `index.html`로 올리고 Pages 활성화 → **`https://silverylaker-cmyk.github.io/VRT1/`**
- 빠른 시험용 대안: Netlify Drop, Cloudflare Pages, 또는 Mac mini에서 `cloudflared`/`ngrok` https 터널.
- 단일 HTML 파일이면 충분(외부 에셋 없음, Three.js는 CDN 로드).

---

## 1. 현재 산출물 (프로토타입 2종)

둘 다 **단일 HTML + Three.js r128(CDN) + 수동 WebXR 세션** 구조. localStorage 미사용.

### A. 후각 재훈련 — 장미 정원 (`rose-garden-olfactory.html`)
- 석양 장미 정원 장면 + **흡입 안내 타이머**(준비 6s → 흡입 15s → 휴식 10s, 3회 반복 → 완료).
- 호흡 안내 원(확장/수축), 단계 전환 시 부드러운 신호음(Web Audio).
- 향은 **실물 키트**(사용자 보유)로 환자가 직접 흡입. 앱은 ①장면 ②타이밍/호흡 큐 담당.
- 자가평가 기록은 **현재 미포함**(사용자 선택). 추후 옵션.
- 컨트롤러 없이 자동 진행, 핀치(select)로 진행/재시작. 화면 미리보기(드래그) 지원.
- 비주얼은 절차적 임시 에셋 → **현실감 업그레이드 = 고해상 360° 사진/포토그래메트리로 스카이박스 교체**.

### B. 전정재활 — 시각 자극 노출 (`index.html` = VRT1 배포본, 원본 `vestibular-vr.html`)
- 두 장면: **마트 통로**(빽빽한 선반 사이를 전진하는 광학 흐름) / **옵토키네틱 필드**(전 시야 점 회전).
- 임상가용 사전 설정: 자극 속도(0~100%), 시각 밀도(낮음/보통/높음), 옵토키네틱 방향(좌우/상하/회전), 세션 길이(1~10분).
- 안전 옵션: 중앙 응시점(난이도↓), 점진적 시작(램프업).
- **안전 핵심: 핀치(select) = 즉시 정지** → 자극 페이드아웃 + 화면 디밍 + "정지됨". 재핀치로 재개.
- HUD: 남은 시간·현재 강도%·"어지러우면 핀치로 정지" 상시 표시.
- 화면에 진단 줄 표시: 프로토콜·보안컨텍스트·임베드여부·immersive-vr/ar 지원·세션 시도 로그.

---

## 2. WebXR / Three.js 기술 메모 (재현 주의점)

- **세션 요청 패턴(검증됨):** 옵션 없이 `immersive-vr`부터, 실패 시 `{optionalFeatures:['hand-tracking']}`, 그다음 `immersive-ar` 순으로 폴백. `local-floor`/`bounded-floor`는 Galaxy XR에서 거부될 수 있으니 **요청하지 말 것**.
- **참조 공간:** `renderer.xr.setReferenceSpaceType('local')` (좌식 기준, 호환성 최고). `local-floor`는 피한다.
- **XR에서 카메라 이동 금지:** XR 중엔 헤드셋이 카메라 포즈를 제어하므로 `camera.position`을 직접 바꿔도 무시됨. **대신 월드(콘텐츠 그룹)를 이동/회전**시켜 광학 흐름(vection)을 만든다. (마트 통로는 `world.position.z += v*dt` + 주기 wrap, 옵토키네틱은 그룹 rotation.)
- **HUD/오버레이:** 캔버스 텍스처 평면을 **`camera`의 자식**으로 추가하면 시야에 고정되어 따라온다. `scene.add(camera)` 필수.
- **입력:** 컨트롤러 없이 `session.addEventListener('select', ...)` = 핀치/탭으로 통일.
- **루프:** XR 진입 시 `renderer.setAnimationLoop(render)`, 비-XR 미리보기는 `requestAnimationFrame`.
- **성능:** 상품은 `InstancedMesh`로(밀도 단계별 인스턴스 수 조절). 모바일급 GPU 고려해 과밀 주의.
- **끊김 없는 통로 루프:** 콘텐츠를 주기 `PERIOD`로 반복 생성하고 `world.z`를 `PERIOD`로 wrap → 이음매 안 보임.

---

## 3. 임상 설계 파라미터

### 후각(표준 Hummel 프로토콜 기반)
- 향 4종: 장미·유칼립투스·레몬·정향. 향마다 장면 매칭(장미→정원, 유칼립투스→숲/스파, 레몬→과수원, 정향→향신료 시장).
- 흡입 ~10–15초 × 수회, 하루 2회, 12주. "냄새 맡으며 대상 시각화"가 핵심 → VR 장면이 후각-시각 연합 강화.
- 추가 커플링 후보(정서/문화 친숙도 높음): 커피→카페, 바닐라/빵→베이커리, 편백/소나무→편백 숲, 라벤더→라벤더 밭, 녹차→녹차밭, 백단향→산사. (실물 향 구득 용이성·삼차신경 자극취 회피 고려.)

### 전정(시각성 어지럼 습관화)
- 치료 성분 = **통제된 광학 흐름 + 시각 복잡도의 점진적 등급화**(사실감 아님).
- 낮은 강도→점진 증가. 앉은 자세 기본, 보조자 동반, 즉시 정지, 금기군(급성 전정신경염·낙상 고위험) 선별.
- 근거: 시각성 어지럼/PPPD에 마트·복잡 환경 노출 습관화는 정식 기법. VR이 잔존 BPPV 증상·균형·불안에서 기존 운동요법 대비 우월 보고 있음.

---

## 4. ENT WebXR 앱 로드맵 (후보)

1. 전정재활(시각 노출) — **진행 중(VRT1)**
2. 후각 재훈련 — **진행 중(장미)**
3. 시선안정화(VOR) 훈련
4. BPPV 이석정복 가이드 + 3D 이석 교육
5. 이명 사운드테라피(주파수·강도 맞춤 + 몰입 환경, 공간음향)
6. 시술 중 몰입 분산(외래 비강 내시경·진정 시술 — 미다졸람 절감 근거 있음, septoplasty 직접 연구는 없음)
7. 공간 청취 훈련(보청기/인공와우)
8. 3D ENT 해부 환자교육

---

## 5. 다음 작업(TODO)

### 즉시
- [ ] `index.html`(전정)을 VRT1에 올리고 Pages 활성화 → 헤드셋에서 https 접속·권한 허용·진입 확인.
- [ ] 진입 성공 후 진단 줄(`프로토콜:https / 보안컨텍스트:예`) 확인.

### 전정 앱 확장
- [ ] 마트 통로에 지나가는 사람(군중) 추가 → 시각 복잡도↑.
- [ ] 옵토키네틱 줄무늬(드럼) 패턴 옵션 추가.
- [ ] 세션 종료 후 어지럼 0~10 자가평가 기록(https 배포 시 localStorage/IndexedDB 사용 가능).

### 후각 앱 확장
- [ ] 장미 외 향 장면 추가(유칼립투스·레몬·정향), 동일 구조 복제.
- [ ] 360° 실사 에셋으로 현실감 업그레이드.
- [ ] (옵션) 강도 자가평가·세션 기록.

### 공통 인프라
- [ ] 앱 공통 셸(장면 로더 + 타이머/HUD/안전정지 모듈) 추출해 재사용.
- [ ] 기록 데이터 저장 방식 확정(로컬 우선, 개인정보보호법 고려).

---

## 6. 레포/배포 메모
- WebXR 배포 레포: `silverylaker-cmyk/VRT1` → `https://silverylaker-cmyk.github.io/VRT1/`
- 기존 관련 레포: `silverylaker-cmyk/AAC`(AAC PWA).
- 단일 HTML이면 루트 `index.html`로. 여러 앱이면 하위 폴더(`/olfactory/`, `/vestibular/`)로 분리 후 각 `index.html`.
