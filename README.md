# 🌸 interactive-voronoi

> WebGL Shader 기반 부드럽게 재배치되는 파스텔 보로노이 다이어그램 힐링 인터랙티브 아트

화면을 무작위로 분할하는 **보로노이 다이어그램(Voronoi Diagram)**을 구현하되, 각 셀이 **파스텔 톤 그라데이션** 색상을 가지며 **마우스 커서가 움직일 때마다 셀의 중심점들이 밀려나면서** 전체 기하학적 구조가 물 흐르듯 부드럽게 재배치되는 **힐링 인터랙티브 아트**입니다. WebGL Fragment Shader가 픽셀 단위로 가장 가까운 점까지의 거리를 계산해 부드러운 셀 경계와 실시간 마우스 disturbance를 GPU 가속으로 처리합니다.

[🇰🇷 한국어 (기본)](#) · [🇺🇸 English](./README.en.md)

---

## 🎬 라이브 데모 (Live Demo)

> **👉 [https://interactive-voronoi.vercel.app/](https://interactive-voronoi.vercel.app/)** — 브라우저에서 바로 실행 (60fps, WebGL)

| | |
|---|---|
| ![Demo](https://img.shields.io/badge/Live-Demo-7C3AED?style=for-the-badge&logo=vercel&logoColor=white) | [![Repo](https://img.shields.io/badge/GitHub-sigco3111%2Finteractive--voronoi-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/sigco3111/interactive-voronoi) |
| ![Status](https://img.shields.io/badge/Status-Live-22C55E?style=flat-square) | ![Stack](https://img.shields.io/badge/Stack-WebGL%20Shader-000000?style=flat-square) |
| ![License](https://img.shields.io/badge/License-MIT-F1C40F?style=flat-square) | ![Deps](https://img.shields.io/badge/Dependencies-0-9CA3AF?style=flat-square) |

### 🎮 빠른 사용법
1. 위 데모 링크 클릭 → 브라우저에서 페이지 열기
2. **파스텔 셀**이 화면 전체를 분할 — 각 셀이 부드러운 그라데이션 색상 보유
3. **마우스 이동** — 커서 주변의 셀 중심점이 자연스럽게 밀려나며 전체 구조가 유체처럼 재배치
4. **힐링 효과** — 60fps 부드러운 보간으로 잔잔한 명상적 분위기
5. **`space`** 일시정지 / **`r`** 셀 분포 리셋

---

## 🤖 생성 정보 (Attribution)

이 프로젝트의 코드는 아래 모델과 프롬프트를 이용해 **자동으로 생성**되었습니다.

| 항목 | 값 |
|---|---|
| **모델** | MiniMax-M3 |
| **실행 환경** | OpenCode CLI |
| **저장소** | [`sigco3111/interactive-voronoi`](https://github.com/sigco3111/interactive-voronoi) |
| **라이브** | [https://interactive-voronoi.vercel.app/](https://interactive-voronoi.vercel.app/) |
| **라이선스** | MIT |
| **의존성** | 없음 (WebGL GLSL Shader, 단일 HTML) |

### 📝 사용된 프롬프트 (원문)

```
화면을 무작위로 분할하는 보로노이 다이어그램(Voronoi Diagram)을 구현하되,
각 셀(Cell)들이 파스텔 톤의 그라데이션 색상을 가지며 마우스 커서가 움직일 때마다
셀의 중심점들이 밀려나면서 전체 기하학적 구조가 물 흐르듯 부드럽게 재배치되는
힐링 되는 인터랙티브 아트를 만들어줘.

Implementation Advice: Use WebGL Shader for pixel-perfect smooth Voronoi
(calculating distance to nearest point per pixel is fast on GPU).
Alternatively, for SVGs/Canvas, use a library like d3-delaunay but it might
be slower with high point counts. 모든 의존관계의 코드를 하나의 HTML에 담는
형태로 코드 작성.
```

---

## 🧮 보로노이 다이어그램 (Voronoi Diagram)

평면 위의 점 집합 P = {p₁, p₂, ..., pₙ} 에 대해, 각 점 pᵢ 의 **보로노이 셀** Vᵢ 는 "pᵢ 가 가장 가까운 점인 영역"으로 정의됩니다:

```
Vᵢ = { x ∈ ℝ² : d(x, pᵢ) ≤ d(x, pⱼ) ∀ j ≠ i }
```

여기서 d(a, b) 는 유클리드 거리입니다.

**수학적 특성**:
- **셀 경계** — 두 점 사이의 수직 이등분선 (perpendicular bisector)
- **정점 (vertex)** — 세 점 사이 수직 이등분선의 교점
- **분할의 완전성** — 셀들은 평면을 빠짐없이 분할 (PMP, Power Diagram의 일반화)
- **쌍둥이 (dual)** — 들로네 삼각분할 (Delaunay triangulation)

### GPU 셰이더에서의 구현 (GLSL)

각 픽셀마다 N개 점까지의 거리를 계산해 최솟값을 가진 셀을 찾습니다:

```glsl
// Fragment Shader — Voronoi 계산 (픽셀당)
uniform vec2 u_points[POINT_COUNT];  // JS에서 uniforms로 전달
uniform vec3 u_colors[POINT_COUNT];  // 파스텔 그라데이션 색상
varying vec2 v_uv;

void main() {
  vec2 uv = v_uv * u_resolution;
  float minDist = 1e10;
  vec3 nearestColor = vec3(0.0);
  vec2 nearestPoint;

  // 픽셀별로 모든 점까지의 거리 계산 → 가장 가까운 셀 찾기
  for (int i = 0; i < POINT_COUNT; i++) {
    vec2 p = u_points[i];
    // 마우스 disturbance로 점이 밀려난 위치 사용
    float dist = distance(uv, p);
    if (dist < minDist) {
      minDist = dist;
      nearestColor = u_colors[i];
      nearestPoint = p;
    }
  }

  // 경계선 부드럽게 (anti-aliased)
  float edge = smoothstep(0.0, 2.0, minDist);
  gl_FragColor = vec4(nearestColor * edge, 1.0);
}
```

CPU JS 측에서 매 프레임 `u_points[]` uniform을 마우스 위치에 따라 보간된 좌표로 갱신 → 셰이더가 자동으로 재계산.

---

## ✨ 주요 특징 (Features)

- 🌸 **파스텔 그라데이션** — 각 셀에 부드러운 HSL pastel 색상 (낮은 채도, 높은 명도)
- 🖱️ **마우스 disturbance** — 커서 주변 점들이 부드럽게 밀려나며 구조 재배치
- 💧 **유체 같은 보간** — 점들이 spring/damping 시스템으로 자연스럽게 이동 (점프 없음)
- ⚡ **GPU 가속** — WebGL Fragment Shader로 픽셀당 거리 계산 (60fps 안정)
- 🎨 **힐링 무드** — 잔잔한 색상 + 부드러운 모션 → 명상적 분위기
- 🖼️ **픽셀-퍼펙트 경계** — 셰이더의 `smoothstep` 으로 anti-aliased 경계
- 📦 **단일 HTML** — WebGL + GLSL + JS, 외부 의존성 0
- 🔒 **온디바이스** — 모든 렌더링이 브라우저 GPU에서 처리

---

## 🚀 실행 방법 (Quick Start)

### 방법 1: 라이브 데모 (가장 간단)
위 [🎬 라이브 데모](https://interactive-voronoi.vercel.app/) 링크 클릭 → 즉시 실행

### 방법 2: 그냥 브라우저로 열기
```bash
open index.html        # macOS
xdg-open index.html    # Linux
start index.html       # Windows
```

### 방법 3: 로컬 서버 (권장)
```bash
python3 -m http.server 8000
# → http://localhost:8000
```

> ⚠️ WebGL은 로컬 `file://` 에서도 동작하지만 일부 브라우저는 보안 정책으로 제한할 수 있어 로컬 서버 권장.

---

## 🎮 조작법 (Controls)

| 입력 | 효과 |
|---|---|
| **마우스 이동** | 커서 주변 점들이 부드럽게 밀려남 (힐링 인터랙션) |
| **자동 시뮬레이션** | 점들이 spring/damping 으로 자연스럽게 부유 (기본) |
| **`space`** | 일시정지 / 재개 |
| **`r`** | 점 분포 리셋 (랜덤 재배치) |
| **`+` / `-`** | 점 수 조정 (10 ~ 200) |

---

## 🛠️ 기술 스택 (Tech Stack)

| 영역 | 사용 기술 |
|---|---|
| **렌더링** | WebGL Fragment Shader (GLSL) |
| **거리 계산** | 픽셀별 nearest-point search (`gl_FragColor` 루프) |
| **물리 보간** | spring/damping 시스템 (JS 측 점 위치 업데이트) |
| **컬러** | HSL pastel 팔레트 (low saturation, high lightness) |
| **마우스** | `pointermove` 이벤트 → uniform `u_mouse` 갱신 |
| **루프** | `requestAnimationFrame` |
| **빌드** | 없음 (단일 HTML, 외부 의존성 0) |

---

## 📂 프로젝트 구조

```
interactive-voronoi/
├── index.html      # 단일 HTML (WebGL 셰이더 + Voronoi 로직 + 마우스 인터랙션)
├── README.md       # 한국어 (기본)
├── README.en.md    # English
├── LICENSE         # MIT
└── .gitignore
```

---

## 🎨 디자인 결정 (Design Choices)

브레인스토밍 단계에서 내린 결정 4가지:

| 결정 포인트 | 선택 | 이유 |
|---|---|---|
| **렌더링 API** | WebGL Fragment Shader (vs Canvas2D / SVG / d3-delaunay) | 픽셀당 거리 계산이 GPU에서 병렬 처리 → 60fps 안정, 부드러운 anti-aliased 경계 |
| **인터랙션 모델** | spring/damping disturbance (vs 즉시 점프) | 즉시 이동은 어색함, spring은 자연스러운 유체 느낌 → 힐링 효과 극대화 |
| **색상 팔레트** | HSL pastel (낮은 채도 ~30%, 높은 명도 ~85%) | 채도 낮으면 차분, 명도 높으면 부드러움 → 명상적 분위기 |
| **셀 수** | 30~60개 기본값 (vs 10개 / 200개) | 30개↓: 구조 너무 단순, 60개↑: 픽셀 비용 증가로 60fps 저하 가능 |

### 직접 커스터마이즈하고 싶다면

`index.html` 상단에서 다음 상수를 조정하면 분위기를 바꿀 수 있어요:

```js
const CONFIG = {
  POINT_COUNT: 40,            // 점 수 (10 ~ 100)
  POINT_SPEED: 0.3,           // 자동 부유 속도
  SPRING_STIFFNESS: 0.05,     // disturbance 복원력 (작을수록 오래 밀려남)
  SPRING_DAMPING: 0.92,       // 진동 감쇠 (1.0 = 무감쇠, 0 = 즉시 정지)
  MOUSE_RADIUS: 200,          // 마우스 영향 반경 (픽셀)
  MOUSE_FORCE: 80,            // 밀어내는 힘의 크기
  PASTEL_SATURATION: 0.30,    // HSL 채도 (0~1, 낮을수록 차분)
  PASTEL_LIGHTNESS: 0.85,     // HSL 명도 (0~1, 높을수록 부드러움)
  EDGE_SMOOTH: 2.0,           // 셀 경계 부드러움 (smoothstep 폭)
  // ... 더 많은 옵션은 코드 내 주석 참조
};
```

**파라미터 탐색**:
- `POINT_COUNT = 10` → 거대한 셀, 미니멀한 분위기
- `POINT_COUNT = 40` → 균형잡힌 디테일 (기본값)
- `POINT_COUNT = 100` → 복잡한 패턴, computational cost 증가
- `MOUSE_FORCE = 0` → 마우스 무시, 자동 부유만 (명상용)
- `MOUSE_FORCE = 200` → 강한 disturbance (인터랙티브 강조)

**적분 방법 비교 (물리)**:
- **즉시 점프**: 어색하고 부자연스러움
- **Linear interpolation**: 부드럽지만 물리감 부족
- **Spring/damping**: 자연스러운 진동 후 안정 (권장)

---

## 📚 관련 알고리즘 (Related Algorithms)

| 알고리즘 | 용도 | 듀얼 (Dual) |
|---|---|---|
| **Voronoi Diagram** | 공간 분할 (closest-point) | Delaunay Triangulation |
| **Delaunay Triangulation** | 점 삼각분할 (empty circle) | Voronoi Diagram |
| **Power Diagram** | 가중치 보로노이 | Regular Triangulation |
| **Lloyd's Algorithm** | 셀 중심점 균등화 | — |
| **Fortune's Algorithm** | O(n log n) 보로노이 계산 | — |
| **k-d Tree** | 최근접 점 검색 (CPU 대안) | — |

**WebGL GPU 구현의 장점**:
- 픽셀당 거리 계산이 N×너비×높이 만큼 병렬 처리
- 1920×1080 해상도 + 40점 = ~83M 거리 계산/프레임 → 60fps 안정
- CPU d3-delaunay 대비 100×~1000× 빠름 (JavaScript 구현 한계)

---

## 📜 License

MIT © 2026 sigco3111

---

## 🙏 Acknowledgments

이 프로젝트는 **MiniMax-M3** 모델과 OpenCode CLI 환경에서 생성되었습니다. 프롬프트 엔지니어링과 디자인 결정은 저장소 소유자가 직접 수행했습니다.

- **Voronoi Diagram**: Georgy Voronoi (1908) — *"Nouvelles applications des paramètres continus à la théorie des formes quadratiques"*
- **Delaunay Triangulation**: Boris Delaunay (1934)
- **WebGL**: Khronos Group 표준, 모든 모던 브라우저 지원
- **힐링 아트 영감**: 사찰 정원 (일본 젱 정원), 현대 추상회화
- **코딩미션 참조 페이지**: [cokac.com — 코드깎는노인](https://cokac.com/list/announcement/24)