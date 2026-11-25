# 5주차 요약 — 빛과 색 (Phong 조명 모델)

출처: [5주차 강의](https://cg.catholic.ac.kr/~mgchoi/CG/weeks/week05/) · 키워드: 색 인지, 반사, Phong, diffuse·specular·ambient, 법선, directional·point·spot
※ 강의 페이지에 "작성 중, 내용이 바뀔 수 있음"이라고 적혀 있습니다. 수업 전에 원문을 다시 확인하세요.

> **한 줄 요약**: 물체의 색은 "어떤 빛이 와서 무엇이 되돌아갔나"의 결과다. Phong 모델은 이를 **ambient + diffuse + specular** 세 항의 합으로 근사한다. 법선 N과 내적 하나로 **밤낮 경계**가 생긴다.

---

## 1. 우리가 색을 보는 원리

- 눈에는 수용체가 **3종류**(S·M·L ≈ 파랑·초록·빨강)뿐입니다. 그래서 빛은 결국 숫자 3개로 줄어들고, 셰이더도 색을 `vec3`(RGB)로 다룹니다.
- 물체는 빛을 내지 않고 **되돌려 보낼 뿐**입니다. 사과가 빨간 것은 빨강 외의 파장을 흡수했기 때문입니다. 그래서 셰이더에서 `물체색 × 빛색`처럼 **곱**합니다.
- 보이는 모습을 정하는 세 가지
  - **어떤 파장**을 반사하나 → 물체색
  - **얼마나** 반사하나 → 밝기
  - **어느 방향**으로 반사하나 → 광택(금속) vs 무광(종이)
- 표면에서 바로 튕긴 빛은 **정반사(specular)** 로, 광원의 색을 그대로 띱니다. 그래서 플라스틱의 하이라이트는 흰색입니다.
- 표면 속으로 들어갔다 나온 빛은 **확산 반사(diffuse)** 로, 사방으로 퍼지며 물체의 색을 띱니다.

## 2. Phong 반사 모델

$$I = k_a I_a + k_d I_d\max(N\cdot L,0) + k_s I_s\max(R\cdot V,0)^{\alpha}$$

| 항 | 흉내 내는 것 | 필요한 벡터 | 끄면 |
|---|---|---|---|
| ambient | 다른 곳에서 튕겨 온 빛 전부 (상수) | 없음 | 그늘이 새까매짐 |
| diffuse | 사방으로 퍼지는 확산광 | **N, L** | 입체감 사라짐 (원이 납작해 보임) |
| specular | 표면에서 바로 튕긴 하이라이트 | **N, L, V** | 전부 무광으로 보임 |

- N(법선), L(광원 쪽), V(눈 쪽), R(L의 반사 방향)은 모두 **단위벡터**여야 하고 **같은 좌표계(월드)** 에 있어야 합니다. 조명 버그의 절반이 이 좌표계 문제입니다.
- **diffuse는 보는 방향과 무관**합니다(람베르트 코사인 법칙: 밝기 ∝ cosθ = N·L).
- **specular는 보는 방향에 의존**합니다. 카메라를 돌리면 하이라이트만 움직이므로 `uViewPos`를 꼭 갱신해야 합니다.
- `shininess`(α)가 크면 하이라이트가 작고 날카로워집니다(2~8 천, 16~32 나무, 64~128 도자기, 200+ 금속).
- ambient는 보통 0.05~0.15이고, **우주 장면은 0~0.03**입니다. 우주에는 빛을 튕겨 줄 벽이 없기 때문입니다.

```glsl
#version 300 es
precision highp float;                                   // 실수 정밀도
in vec3 vWorldPos;                                       // 이 픽셀의 월드 위치 (버텍스 셰이더에서 보간됨)
in vec3 vNormal;                                         // 이 픽셀의 법선 (보간됨)
uniform vec3 uLightPos, uLightColor, uViewPos, uBaseColor; // 광원 위치·색, 카메라 위치, 물체색
uniform float uKa, uKd, uKs, uShininess;                 // 세 항의 세기와 광택
out vec4 fragColor;                                      // 최종 색
void main() {
  vec3 N = normalize(vNormal);                           // 보간된 법선은 길이가 1이 아닐 수 있어 다시 정규화
  vec3 L = normalize(uLightPos - vWorldPos);             // 표면 → 광원 방향
  vec3 V = normalize(uViewPos - vWorldPos);              // 표면 → 눈 방향
  vec3 R = reflect(-L, N);                               // 빛의 반사 방향 (입사 방향 −L을 넣어야 함!)
  float lambert = max(dot(N, L), 0.0);                   // cosθ. 빛을 등지면 음수 → 0으로 자름 = 밤
  float spec = lambert > 0.0                             // 빛이 닿는 면에서만
             ? pow(max(dot(R, V), 0.0), uShininess)      //   R과 V가 가까울수록 밝게, 거듭제곱으로 폭 조절
             : 0.0;                                      //   뒷면에 하이라이트가 생기는 버그 방지
  vec3 ambient  = uKa * uBaseColor;                      // 방향과 무관한 바탕 밝기
  vec3 diffuse  = uKd * uBaseColor * uLightColor * lambert; // 물체색을 띠는 확산광
  vec3 specular = uKs * uLightColor * spec;              // 물체색을 곱하지 않음 → 흰 하이라이트
  fragColor = vec4(ambient + diffuse + specular, 1.0);   // 세 항의 합
}
```

### 법선 변환 주의

- 비균등 스케일을 쓰면 모델 행렬을 그대로 곱한 법선은 표면과 수직이 아니게 됩니다. → **법선 행렬 = 모델 행렬의 역전치(inverse transpose)** 를 쓰세요.
- 법선은 방향이므로 이동의 영향을 받으면 안 됩니다(w = 0, 2주차).

## 3. 광원 3종류 — 다른 것은 `L`과 감쇠(att)뿐

| 광원 | L 구하기 | 감쇠 | 예 |
|---|---|---|---|
| directional | `normalize(-uLightDir)`, 모든 픽셀에서 같음 | 없음 | 지표면에서 본 햇빛 |
| point | `normalize(uLightPos - pos)` | $\dfrac{1}{1+k_1d+k_2d^2}$ | 전구, **태양계 장면의 태양** |
| spot | point와 같음 + 원뿔 밖은 잘라냄 | point + 원뿔 경계 | 손전등, 서치라이트 |

```glsl
// point 광원의 L과 감쇠
vec3  d    = uLightPos - vWorldPos;                      // 표면 → 광원 벡터
float dist = length(d);                                  // 광원까지 거리
vec3  L    = d / dist;                                   // 단위 방향
float att  = 1.0 / (1.0 + uLinear*dist + uQuad*dist*dist); // 멀수록 어두워짐 (1/d²은 d→0에서 발산하므로 이 형태 사용)
// spot이면 여기에 추가: 원뿔 축과의 cos으로 부드럽게 자르기
float c = dot(-L, normalize(uSpotDir));                  // 원뿔 축과 이루는 각의 cos (acos 대신 cos끼리 비교해서 계산이 쌈)
att *= clamp((c - uCosOuter) / (uCosInner - uCosOuter), 0.0, 1.0); // 안쪽 100% → 바깥 0%
```

- 태양 자신은 조명 계산 없이 밝은 색을 그대로 출력합니다(자체발광). 예: `if (uEmissive) { fragColor = vec4(uBaseColor, 1.0); return; }`

## 4. 과제 4 — 태양 하나로 만드는 낮과 밤 (요약)

**실습 1** (`task1.html`, 출발 파일 `w5-start.html`의 `★ 단계 N` 자리를 채움)

| 단계 | 할 일 | 확인 |
|---|---|---|
| 1 | diffuse 추가 | 원이 구로 보임 (`max` 제거 시 결과도 예측해 보기) |
| 2 | ambient 추가 | 밤 쪽이 완전히 검지는 않음 |
| 3 | specular 추가 | 시점을 돌리면 하이라이트만 이동 |
| 4 | 거리 감쇠 | 가까운 공이 더 밝음 |
| 5 | 법선 행렬(`uNormalMat`)로 교체 | 비균등 스케일에서 `mat3(uModel)`과 비교 캡처 2장 |
| 6 | 광원 종류 바꾸기 | 달라진 점을 한 문단으로 |

**실습 2** (`task2.html`, 빛으로 분위기 만들기, 정답 없음)

- 광원 **2개 이상, 서로 다른 종류 2가지 이상**, 광원마다 다른 색을 씁니다(따뜻한 색 vs 차가운 색).
- 물체마다 재질(`shininess`, 물체색)을 다르게 합니다.
- **키보드(1, 2, 3)로 광원을 켜고 끌 수 있게** 합니다.
- 광원이 여러 개이면 `shade()`를 광원마다 부르고 더합니다. **ambient는 한 번만** 더합니다.
- 예시 코드(`w5-example.html`)를 그대로 내거나 색만 바꿔 내면 점수가 없습니다.

**마감**: 6주차 목요일 수업 전 · 제출 위치는 `week5/{task1.html, task2.html, week5.md, images/}`입니다.

## 5. 자주 하는 실수

| 증상 | 원인 |
|---|---|
| 광원을 옮겨도 하이라이트가 안 움직임 / 카메라를 돌리면 빛이 따라 돎 | 위치 벡터들의 **좌표계가 섞임** (모두 월드 좌표로) |
| 뒷면에 하이라이트 | `lambert > 0` 검사 없이 specular 계산 |
| 하이라이트 위치가 이상함 | `reflect(L, N)`으로 씀 → `reflect(-L, N)`이 맞음 |
| 늘린 물체의 조명이 이상함 | 법선에 모델 행렬을 그대로 곱함 → 역전치 사용 |
| 얼룩진 조명 | 보간된 법선을 다시 `normalize` 하지 않음 |
| 전체가 뿌옇게 들뜸 | ambient가 너무 크거나 광원 수만큼 여러 번 더함 |
