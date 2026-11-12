# Three.js 디지털 트윈 프로젝트 분석 및 문법 구조 가이드

본 문서는 첨부된 두 개의 단일 HTML 기반 3D 웹 애플리케이션(**스마트 학교 교실 디지털 트윈**, **한국형 전기 시내버스 스마트팩토리 디지털 트윈**)의 실행 방법과 전체적인 아키텍처 및 문법 구조를 분석한 가이드입니다.

---

## 1. 프로젝트 개요

두 파일 모두 외부 3D 모델 파일(`.glb`, `.gltf`)이나 외부 이미지 텍스처를 불러오지 않고, **Three.js의 기본 기하학 도형(Box, Cylinder, Sphere, Tube)과 HTML5 `<canvas>` 2D 동적 텍스처만을 조합(Procedural Modeling)**하여 구현된 교육용 인터랙티브 3D 웹 애플리케이션입니다.

| 구분 | 파일 1: 스마트 학교 교실 | 파일 2: 한국형 시내버스 조립공장 |
| :--- | :--- | :--- |
| **파일명** | `smart_classroom_camera_stable.html` | `ThreeJS_한국_시내버스_조립공장_3D.html` |
| **Three.js 버전** | `v0.164.1` (CDN Import Map) | `v0.180.0` (CDN Import Map) |
| **핵심 주제** | 교실 환경 IoT 센서 시뮬레이션 및 설비 제어 | 8단계 버스 조립 공정 시퀀스 및 부품 분해/검사 |
| **특화 기능** | 카메라 위치에 따른 벽면 자동 절개(Cutaway), JSON/CSV/PNG 내보내기, 실시간 차트 | Raycaster 기반 부품 클릭 선택, X-Ray 반투명 모드, 도장 색상 변경, 도어 개폐 |

---

## 2. 실행 방법

두 파일 모두 **ES Modules (`<script type="module">`)** 및 **CDN(jsDelivr)**을 사용하여 Three.js 라이브러리를 로드합니다.

### 방법 A: VS Code Live Server 확장 프로그램 사용 (가장 권장)
1. Visual Studio Code에서 해당 HTML 파일이 있는 폴더를 엽니다.
2. 마켓플레이스에서 **Live Server** 확장을 설치합니다.
3. HTML 파일 우클릭 후 **`Open with Live Server`**를 클릭하면 로컬 서버(`http://127.0.0.1:5500/...`)가 열리며 3D 화면이 실행됩니다.

### 방법 B: Python 내장 HTTP 서버 사용
터미널(명령 프롬프트)을 열고 파일이 위치한 디렉터리로 이동한 뒤 아래 명령어를 실행합니다.
```bash
# Python 3 기준
python -m http.server 8000
```
이후 웹 브라우저(Chrome, Edge 권장) 주소창에 `http://localhost:8000/파일명.html`을 입력하여 접속합니다.

### ⚠️ 실행 시 주의사항
* **인터넷 연결 필수**: `<script type="importmap">`을 통해 `cdn.jsdelivr.net`에서 Three.js 코어와 `OrbitControls`를 실시간으로 다운로드하므로 네트워크 연결이 필요합니다.
* **로컬 파일 직접 열기(`file://`) 제한**: 최신 브라우저의 보안 정책(CORS)으로 인해 파일을 더블 클릭하여 직접 열면 모듈 로딩이 차단될 수 있으므로 반드시 로컬 웹 서버 환경을 통해 실행해야 합니다.
* **하드웨어 가속**: WebGL 렌더링을 사용하므로 브라우저 설정에서 '하드웨어 가속 사용'이 켜져 있어야 원활하게 작동합니다.

---

## 3. 전체적인 코드 및 문법 구조

두 파일은 외부 CSS나 JS 파일 분리 없이 **단일 HTML 파일(Single-File Architecture)** 내부에 스타일, 마크업, 메타데이터, 3D 렌더링 로직이 모두 통합된 구조를 가지고 있습니다.

```text
📄 Single HTML File
 ├── 1. <head> (메타데이터, CSS 스타일, Import Map, JSON 스펙)
 ├── 2. <body> (3D 캔버스 컨테이너, HUD 오버레이 UI, 모달/로딩 화면)
 └── 3. <script>
      ├── 3-1. 일반 스크립트 (초기화 실패 감지 및 에러 핸들링)
      └── 3-2. 모듈 스크립트 (<script type="module"> : Three.js 핵심 로직)
```

### 3-1. `<head>` 영역: 환경 설정 및 선언적 데이터
1. **CSS Grid & Flexbox 레이아웃**:
   * 교실 예제는 전체 화면 3D 뷰(`position: fixed; inset: 0`) 위에 반투명 글래스모피즘(`backdrop-filter: blur()`) 패널을 띄우는 **HUD(Head-Up Display) 방식**을 사용합니다.
   * 버스 공장 예제는 `display: grid; grid-template-columns: 292px 1fr 272px`를 사용하여 좌측(공정 제어), 중앙(3D 뷰포트), 우측(관제 패널)로 분할된 **대시보드 레이아웃**을 취합니다.
2. **`<script type="importmap">`**:
   * 빌드 도구(Webpack, Vite 등) 없이 브라우저에서 `import * as THREE from 'three'` 문법을 사용할 수 있도록 모듈 경로를 매핑합니다.
3. **JSON 메타데이터 (`smart_classroom` 특화)**:
   * `<script type="application/json" id="sceneSpec">` 태그 내부에 가구 좌표, 카메라 프리셋, 시뮬레이션 수식, 좌표계 변환 규칙(Three.js $\leftrightarrow$ Blender)을 JSON 형태로 내장하여 코드와 데이터를 분리했습니다.

### 3-2. `<body>` 영역: UI 계층 구조
* **3D 스테이지 (`#stage` / `#world`)**: Three.js의 `WebGLRenderer.domElement`(`<canvas>`)가 동적으로 삽입되는 컨테이너입니다.
* **상태 표시 및 제어 패널**: 슬라이더(`<input type="range">`), 시점 전환 버튼(`data-view`), 시나리오 프리셋(`data-scenario`) 등 DOM 요소에 `data-*` 속성을 부여해 이벤트 위임 및 상태 동기화를 간결하게 처리합니다.
* **예외 처리 오버레이 (`#loading`, `#fallback`, `#graphicsNotice`)**: 3D 엔진 로딩 중 화면과 WebGL 컨텍스트 손실(`webglcontextlost`) 또는 네트워크 오류 시 표시할 대체 UI(SVG 다이어그램 및 안내 문구)를 갖추고 있습니다.

### 3-3. `<script type="module">` 영역: Three.js 핵심 문법 패턴

#### ① 렌더러(Renderer), 씬(Scene), 카메라(Camera) 초기화
```javascript
const renderer = new THREE.WebGLRenderer({ antialias: true, preserveDrawingBuffer: true });
renderer.outputColorSpace = THREE.SRGBColorSpace; // 정확한 색감 표현
renderer.toneMapping = THREE.ACESFilmicToneMapping; // 실사풍 명암 대비
renderer.shadowMap.enabled = true; // 그림자 활성화

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true; // 부드러운 관성 회전
```
* **성능 최적화 기법**: `preserveDrawingBuffer: true` 옵션을 켜서 `canvas.toBlob()` / `toDataURL()`을 통한 PNG 스크린샷 저장을 지원하며, 교실 파일에서는 정적 객체의 그림자 갱신(`renderer.shadowMap.autoUpdate = false`)을 제한해 GPU 부하를 낮췄습니다.

#### ② 헬퍼 함수(Factory Pattern)를 통한 절차적 모델링
반복적인 3D 메쉬 생성 코드를 줄이기 위해 커스텀 빌더 함수를 정의하여 계층 구조(`THREE.Group`)를 조립합니다.
* `bx(parent, ...)` / `cyl(...)` / `sphere(...)` / `rod(...)`: 지오메트리(Geometry)와 머티리얼(Material)을 결합해 메쉬를 생성하고 그림자 속성(`castShadow`, `receiveShadow`)을 일괄 적용한 뒤 부모 그룹에 `add()` 합니다.
* `canvasTexture(...)`: 메모리 상에 `<canvas>`를 생성하여 2D 그래픽(전자칠판 UI, 센서 라벨, 버스 노선 번호판, 공장 안전 표지판)을 그린 뒤 `THREE.CanvasTexture`로 변환해 3D 객체 표면에 매핑합니다.

#### ③ 상태 관리(State) 및 인터랙션 로직
* **중앙 상태 객체 (`sim` / `state`)**: 현재 온도, CO₂ 농도, 조립 단계, X-Ray 여부 등 모든 시뮬레이션 변수를 단일 객체로 관리합니다.
* **Raycasting (버스 공장 특화)**: 마우스 클릭(`pointerup`) 시 화면 2D 좌표를 정규화 장치 좌표(NDC, $-1 \sim +1$)로 변환한 뒤 `THREE.Raycaster`를 쏘아 클릭된 3D 메쉬의 `userData.partId`를 역추적하여 부품 설명을 표시합니다.
* **동적 절개(Wall Cutaway - 교실 특화)**: 카메라의 월드 좌표(`camera.position`)가 교실 외벽 바깥으로 이동하면 해당 벽면 메쉬의 `visible` 속성을 `false`로 전환하여 시야 가림을 방지합니다.

#### ④ 애니메이션 루프 (`requestAnimationFrame` / `setAnimationLoop`)
```javascript
function animate() {
  requestAnimationFrame(animate);
  const dt = Math.min(clock.getDelta(), 0.06); // 프레임 드롭 시 급격한 변화 방지
  
  // 1. 물리/환경 수치 보간 (지수 감쇠 함수 활용)
  // 2. 카메라 시점 전환 트윈 (선형 보간 lerpVectors 및 Smoothstep 이징)
  // 3. 부품 이동 및 도어 개폐 애니메이션 (THREE.MathUtils.lerp)
  
  controls.update();
  renderer.render(scene, camera);
}
```
* **프레임 독립적 애니메이션**: `THREE.Clock`의 `getDelta()`로 경과 시간(`dt`)을 계산하여 모니터 주사율(60Hz, 144Hz 등)에 관계없이 동일한 속도로 시뮬레이션이 진행되도록 설계되었습니다.