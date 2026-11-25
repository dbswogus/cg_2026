# 6주차 요약 — PBR과 텍스처 매핑

출처: [6주차 강의](https://cg.catholic.ac.kr/~mgchoi/CG/weeks/week06/) · 키워드: PBR, microfacet, BRDF, Fresnel, roughness, metallic, UV, mipmap, normal map, AO, environment map, multi-pass

> **한 줄 요약**: 같은 모양을 나무·돌·금속으로 보이게 하려면 **재질 값(base color, roughness, metallic, normal)** 을 물리에 가깝게 설명하는 **PBR**을 쓰고, 그 값을 **텍스처**로 위치마다 다르게 준다.

---

## 1. 재질을 설명하는 4가지 값

| 값 | 정하는 것 | 화면 변화 |
|---|---|---|
| base color | 비금속의 확산색 / 금속의 반사색 | 나무의 갈색, 구리의 유색 반사 |
| **roughness** | 표면 미세 요철의 거칠기 | 낮음 = 좁고 선명한 반사, 높음 = 넓고 흐린 반사 |
| **metallic** | 금속(1)인가 비금속(0)인가 | 금속은 확산이 없고 반사가 색을 띰 |
| normal | 각 지점의 조명용 방향 | 흠집·돌기 같은 명암 |

- 깨끗한 금속은 metallic 1, 나무·돌·플라스틱은 0입니다. **녹과 페인트는 금속 위에 있어도 비금속(0)** 입니다.
- 중간값(0.5 등)은 "반짝임 세기 조절"이 아니라 경계에서 섞이는 경우에만 생깁니다.

## 2. 자연현상 → 모델

| 관찰 | 모델 | PBR 식의 항 |
|---|---|---|
| 평평해 보여도 확대하면 작은 면들이 제각각 → 거칠수록 하이라이트가 넓음 | **미세면(microfacet)** 방향 분포 | **D** (분포) |
| 발밑 물은 바닥이 보이고, 먼 물은 하늘이 비침 (스치는 각도일수록 반사 증가) | **프레넬(Fresnel)** | **F** (반사율) |
| 옆 미세면이 빛이나 시선을 가림 (shadowing / masking) | 가림 | **G** (기하 감쇠, 바닥 그림자와는 다름) |
| 받은 빛보다 더 많이 반사할 수 없음 | **에너지 보존** | 확산에 `(1−F)(1−metallic)`만 배분 |

- **BRDF**: L 방향에서 온 빛이 V 방향으로 얼마나 나가는지를 나타내는 함수입니다. 방향별 분포라 값이 1을 넘을 수도 있습니다.

## 3. PBR 공식과 셰이더 (Cook-Torrance / GGX)

$$H=\frac{L+V}{\lVert L+V\rVert},\quad f_r=\underbrace{\frac{(1-F)(1-m)\,c}{\pi}}_{\text{확산}}+\underbrace{\frac{D\,F\,G}{4(N\cdot L)(N\cdot V)}}_{\text{정반사}}$$

```glsl
const float PI = 3.14159265;                                  // 원주율
vec3 directPBR(vec3 N, vec3 V, vec3 L, vec3 radiance,         // 법선, 눈 방향, 광원 방향, 광원 세기·색
               vec3 baseColor, float roughness, float metallic) { // 재질 값
  float nv = max(dot(N, V), 0.0);                             // 눈과 표면의 cos
  float nl = max(dot(N, L), 0.0);                             // 빛과 표면의 cos (람베르트)
  if (nv <= 0.0 || nl <= 0.0) return vec3(0.0);               // 보이지 않거나 빛이 안 닿으면 0
  vec3 H = normalize(V + L);                                  // 하프벡터: L을 V로 반사하는 미세면의 법선
  float r = clamp(roughness, 0.045, 1.0);                     // 0이면 수식이 발산하므로 최소값 지정
  float a = r * r, a2 = a * a;                                // GGX는 roughness의 제곱을 씀
  float nh = max(dot(N, H), 0.0);                             // 미세면 법선과 표면 법선의 cos
  float d = nh * nh * (a2 - 1.0) + 1.0;                       // D 계산의 분모 일부
  float D = a2 / max(PI * d * d, 1e-7);                       // D: H를 향한 미세면이 얼마나 많은가 (GGX)
  vec3 F0 = mix(vec3(0.04), baseColor, metallic);             // 정면 반사율: 비금속 4%, 금속은 자기 색
  vec3 F = F0 + (1.0 - F0) * pow(1.0 - max(dot(H, V), 0.0), 5.0); // F: Schlick 근사, 스칠수록 반사 증가
  float k = (r + 1.0) * (r + 1.0) / 8.0;                      // G 근사용 계수 (직접광용)
  float G = (nv / (nv*(1.0-k)+k)) * (nl / (nl*(1.0-k)+k));   // G: 눈 쪽·빛 쪽 가림 (Smith-Schlick)
  vec3 specular = D * F * G / max(4.0 * nv * nl, 1e-6);       // 정반사 BRDF
  vec3 kd = (1.0 - F) * (1.0 - metallic);                     // 반사하고 남은 빛만 확산으로 (금속은 확산 0)
  return (kd * baseColor / PI + specular) * radiance * nl;    // (확산 + 정반사) × 빛 × cos
}
```

- 수치 예: 비금속의 F는 정면에서 0.04(4%)이고, 85°로 스칠 때는 약 0.65(65%)까지 올라갑니다.

| | Phong (5주차) | PBR (6주차) |
|---|---|---|
| 재질 값 | kd, ks, shininess (물리적 의미 약함) | base color, roughness, metallic |
| 각도별 반사율 | 일정 | 프레넬 F |
| 에너지 보존 | 보장 안 함 | 확산과 정반사를 연계 |
| 장점 | 식이 짧고 가벼움 | 조명이 바뀌어도 같은 재질 값을 재사용하기 좋음 |

## 4. 텍스처 매핑

- **UV** = 이미지 위의 주소(0~1)입니다. 정점에 UV를 주면 삼각형 안에서 보간되고, 픽셀마다 `texture(img, uv)`로 읽습니다. 정점보다 훨씬 촘촘한 정보를 담을 수 있습니다.
- 범위 밖 UV 처리: `REPEAT`(반복, 타일), `CLAMP_TO_EDGE`(가장자리 유지, 라벨), `MIRRORED_REPEAT`(뒤집어 반복)
- 해상도를 2배로 하면 메모리는 4배입니다(4096² RGBA8 = 64 MiB, 밉맵 포함 약 4/3배).

```js
async function loadTexture(gl, url, isColor) {                // 이미지 파일을 GPU 텍스처로
  const image = new Image();                                  // 브라우저 이미지 객체
  image.src = url;                                            // 파일 경로 (file:// 말고 HTTP 서버로 실행)
  await image.decode();                                       // 다 읽힐 때까지 기다림
  const tex = gl.createTexture();                             // GPU 텍스처 생성
  gl.bindTexture(gl.TEXTURE_2D, tex);                         // 작업 대상으로 선택
  gl.pixelStorei(gl.UNPACK_FLIP_Y_WEBGL, true);               // 이미지 위아래와 UV의 v축 방향 맞추기
  gl.texImage2D(gl.TEXTURE_2D, 0,                             // 밉 레벨 0(원본)에
    isColor ? gl.SRGB8_ALPHA8 : gl.RGBA8,                     //   색 이미지는 sRGB(자동 선형화), 수치 데이터는 그대로
    gl.RGBA, gl.UNSIGNED_BYTE, image);                        //   입력 형식과 이미지
  gl.generateMipmap(gl.TEXTURE_2D);                           // 1/2, 1/4… 축소본 자동 생성
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR_MIPMAP_LINEAR); // 축소: trilinear
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);               // 확대: bilinear
  return tex;                                                 // 완성된 텍스처
}                                                             // 셰이더에서는 texture(uMap, vUV).rgb 로 읽음
```

### 필터링 — 픽셀 하나가 덮는 텍셀 영역에 맞추기

| 방법 | 동작 | 용도 / 한계 |
|---|---|---|
| NEAREST | 가장 가까운 텍셀 1개 | 픽셀아트. 축소하면 모아레(aliasing) 발생 |
| LINEAR | 주변 4텍셀 평균 | 확대 시 부드러움. 많이 축소하면 부족 |
| Mipmap + Trilinear | 미리 줄인 레벨 2개를 섞음 | 멀리 있는 무늬의 깜빡임 감소 (메모리 +33%) |
| Anisotropic | 길쭉한 영역을 따라 여러 번 샘플 | 비스듬한 바닥이 선명해짐 (확장 기능) |

## 5. 색 이외의 맵

| 맵 | 저장 내용 | 주의 |
|---|---|---|
| Base color | 표면색 | **sRGB → 선형으로 변환** 후 계산 |
| Normal | 접선 공간 법선 (평평하면 보라색 (0.5,0.5,1)) | `rgb*2−1`로 복원 후 **TBN 행렬**로 변환. 실루엣은 안 바뀜 |
| Roughness / Metallic | 위치별 수치 | 선형 데이터라 sRGB 변환 금지 |
| AO | 틈새에서 환경광이 가려지는 정도 | **간접광에만** 곱함 (직접광에 곱하지 않음) |
| Emission | 스스로 내는 빛 | 주변을 비추지는 않음 |
| **ARM** | R = AO, G = Roughness, B = Metallic을 한 장에 | 제공자의 채널 약속을 확인 |

```glsl
vec3 arm = texture(uARM, vUV).rgb;                           // 한 번 읽어서 세 값 얻기 (선형 데이터)
float ao = arm.r;                                            // R: 환경광 가림
float roughness = clamp(arm.g * uRough, 0.045, 1.0);         // G: 거칠기 (최소값 보장)
float metallic = arm.b * uMetal;                             // B: 금속 여부
vec3 nTS = texture(uNormalMap, vUV).xyz * 2.0 - 1.0;         // 0~1 색 → −1~1 방향
vec3 N = normalize(mat3(T, B, geomN) * nTS);                 // 접선 공간 → 월드 (T·B·N을 열로)
vec3 color = directPBR(N, V, L, radiance, base, roughness, metallic) // 직접광
           + indirect * ao;                                  // + AO로 줄인 간접광
```

- **마스크**: 0~1 값으로 두 재질을 섞습니다. 예: `mix(rock, ice, mask)`. 예를 들어 `abs(normalize(vLocal).y)`(적도 0 → 극 1)를 `smoothstep`에 넣으면 극지방 얼음을 만들 수 있습니다.
- **환경맵**: 주변 풍경을 방향으로 읽습니다. `texture(uCubeEnv, reflect(-V, N))`은 반사, `refract(-V, N, 1.0/1.5)`는 유리 굴절입니다.
- **색 처리 순서**: sRGB 텍스처 → 선형 → 조명 계산 → HDR → 톤 매핑 → 화면용 sRGB. 변환은 **한 번만** 합니다.

## 6. Multi-pass

- 한 번 그린 결과를 텍스처(FBO)에 저장하고 다음 패스에서 다시 읽어 가공합니다. 사진에 필터를 거는 것과 비슷합니다.
- **Emission**: 표면이 밝게 빛남. multi-pass가 필요 없습니다.
- **Bloom**: 밝은 부분을 주변 픽셀로 번지게 함. 이웃 픽셀을 읽어야 하므로 **별도 패스가 필요**합니다.
- **Tone mapping**: 1보다 큰 HDR 밝기를 화면 범위로 압축. 예: Reinhard `hdr/(1+hdr)`. 마지막 셰이더 안에서도 가능합니다.

## 7. 과제 5 (요약)

**실습 1** — 시작 코드 ZIP의 `task1.js` TODO 1~5 채우기 (`week6`에 풀고 HTTP 서버로 실행)

| TODO | 할 일 | 확인 |
|---|---|---|
| 1 | 색 텍스처 읽기 `base = srgbToLinear(texture(uColor, vUV).rgb) * uTint;` | 큐브에 무늬 |
| 2 | `directPBR()`의 D·F·G 작성, Phong 대체 | roughness·metallic에 따라 반사 변화 |
| 3 | ARM → ao, roughness, metallic | 녹과 금속 부분이 다르게 보임 |
| 4 | 법선맵 + TBN | 흠집 명암 (실루엣은 그대로) |
| 5 | 환경맵 반사 | 직접광을 0으로 해도 반사가 남음 |
| 6 | 나무·금속·돌 비교, 필터링 예제 실행 | 차이를 설명 |

**실습 2** — 나만의 태양계 행성 재질을 설계합니다(마스크 등으로 위치마다 다른 재질).

## 8. 자주 하는 실수

| 실수 | 바로잡기 |
|---|---|
| normal·roughness 맵에 sRGB 변환 | 수치 데이터는 **변환하지 않음** |
| 색 텍스처를 선형화 없이 계산하거나, 두 번 선형화 | `SRGB8_ALPHA8`이면 자동이므로 `pow`를 또 쓰지 않음 |
| 텍스처가 위아래로 뒤집힘 | `UNPACK_FLIP_Y_WEBGL` 확인 |
| `file://`로 열어서 텍스처가 안 보임 | Live Server 또는 `python -m http.server` |
| AO를 직접광에도 곱함 | 간접광에만 |
| 녹 영역까지 metallic = 1 | 녹·페인트는 0 |
