# 3주차 처음부터 정리 — 그래픽스 파이프라인과 셰이더

출처: [3주차 강의 원문](https://cg.catholic.ac.kr/~mgchoi/CG/weeks/week03/)

강의 순서에 맞춰 개념을 쉽게 풀어 쓴 학습 노트입니다. 코드는 이해를 위한 작은 예제이며, 마지막에는 통째로 실행할 수 있는 예제를 넣었습니다. 중간의 GLSL 조각은 셰이더의 `main()` 안에서 사용한다고 생각하면 됩니다.

## 0. 좌표계 — 먼저 위쪽 방향을 약속한다

이번 주부터 **y-up, 오른손 좌표계**를 사용합니다. 2주차의 z-up과 구분하세요.

- x: 오른쪽, y: 위쪽, z: 기본 화면에서 관찰자 쪽 방향으로 생각합니다.
- 오른손 좌표계에서는 `x축 × y축 = z축`입니다.
- 위쪽 축과 손잡이는 별개의 약속입니다. 다른 도구의 모델을 가져오면 변환이 필요할 수 있습니다.

```js
const top = [0, 1, 0];   // y가 양수이므로 위쪽
const right = [1, 0, 0]; // x가 양수이므로 오른쪽
```

WebGL이 모델의 위쪽 축을 강제하지는 않습니다. y-up은 이 수업에서 사용하는 모델·세계 좌표의 약속입니다. 카메라를 움직이면 화면에서 보이는 축 방향도 달라집니다.

## 1. 메시 — 3차원 모델의 실제 모습은 숫자 배열

메시(mesh)는 표면에서 고른 점인 **정점**과, 그 정점을 연결한 **삼각형**으로 표면을 근사한 것입니다. 정점은 단순한 위치뿐 아니라 법선·색·UV 등의 속성을 함께 가질 수 있습니다.

| 데이터 | 뜻 | 정점 하나에 필요한 값 |
|---|---|---|
| 위치 | 어디에 있는가 | x, y, z |
| 법선 | 표면이 어느 방향을 향하는가 | nx, ny, nz |
| 색 | 어떤 색인가 | r, g, b |
| UV | 텍스처 그림의 어디를 읽는가 | u, v |
| 인덱스 | 어느 정점들을 연결하는가 | 삼각형마다 정점 번호 3개 |

```js
// 연속된 숫자 3개가 정점 하나의 위치입니다.
const positions = new Float32Array([
  -0.7, -0.5, 0, // 정점 0: 왼쪽 아래
   0.7, -0.5, 0, // 정점 1: 오른쪽 아래
   0.0,  0.7, 0, // 정점 2: 위쪽
]);

const colors = new Float32Array([
  1, 0, 0, // 정점 0은 빨강
  0, 1, 0, // 정점 1은 초록
  0, 0, 1, // 정점 2는 파랑
]);

// 세 정점 모두 +z 방향을 향하는 평평한 삼각형입니다.
const normals = new Float32Array([
  0, 0, 1,  0, 0, 1,  0, 0, 1,
]);
const indices = new Uint16Array([0, 1, 2]); // 0→1→2를 연결
```

각 배열에서 **같은 정점 번호에 해당하는 값들이 한 묶음**입니다. 인덱스로 정점을 재사용하면 메모리를 줄일 수 있습니다. 다만 정육면체처럼 면마다 법선이 달라야 하면 같은 위치라도 정점을 나눠 저장합니다.

분할을 늘리면 구의 윤곽이 매끄러워지지만 메모리와 계산량도 증가합니다. 화면에서 작은 물체에 지나치게 많은 정점을 쓰면 낭비이므로, 거리에 따라 상세도를 바꾸는 LOD를 사용합니다. WebGL은 삼각형 외에 점과 선도 그릴 수 있지만, 이번 수업의 표면은 삼각형으로 표현합니다.

## 2. CPU와 GPU — 계산을 나누는 이유

**CPU의 JavaScript는 준비하고 지시하며, GPU의 셰이더는 많은 데이터에 같은 계산을 적용합니다.**

| CPU에서 하는 일 | GPU에서 하는 일 |
|---|---|
| 입력 처리, 시간 계산, 장면 구성 | 정점 위치 변환 |
| 메시와 버퍼 준비 | 정점 사이 값 보간 |
| 셰이더 준비, 그리기 명령 | 프래그먼트 색 계산 |

정점 A의 변환은 정점 B의 결과를 기다릴 필요가 없습니다. 이런 독립적인 계산을 많이 동시에 처리하는 것이 GPU의 장점입니다. 셰이더에는 정점 또는 프래그먼트 **하나가 할 일**을 적습니다.

강의의 Tensor Core는 행렬 계산, RT Core는 광선 교차 계산을 돕는 전용 회로의 예입니다. 이번 WebGL2 예제에서는 이 회로들을 직접 제어하지 않습니다.

## 3. 그래픽스 파이프라인 — 숫자가 화면이 되는 순서

```text
CPU: 메시를 GPU 버퍼에 올리고 그리기 명령을 보냄
 ↓
버텍스 셰이더: 정점 위치와 다음 단계로 보낼 값을 계산
 ↓
기본도형 조립: 정점들을 삼각형으로 묶음
 ↓
클리핑 → w로 나누기 → 화면 영역으로 변환
 ↓
래스터화: 삼각형이 덮는 샘플을 찾고 정점 속성을 보간
 ↓
프래그먼트 셰이더: 보간된 값으로 색을 계산
 ↓
깊이 검사·혼합 등 → 프레임 버퍼
```

직접 작성하는 핵심은 **버텍스 셰이더와 프래그먼트 셰이더**입니다. 정점 셰이더는 정점 입력에 대해 실행되고, 프래그먼트 셰이더는 삼각형이 덮는 영역에서 훨씬 많이 실행될 수 있습니다. 따라서 비싼 계산을 프래그먼트마다 하면 비용이 커집니다.

정확히는 프래그먼트와 최종 화면 픽셀이 일대일 대응하지는 않습니다. 겹친 면이 같은 픽셀에 여러 프래그먼트를 만들 수 있고, 깊이 검사에서 탈락할 수도 있습니다. 위 순서는 개념적인 설명이며 깊이 검사가 일찍 수행되는 최적화도 있습니다.

## 4. 하나의 화면을 만드는 프로그램 세 개

1. JavaScript: WebGL 준비, 데이터 전송, 그리기 명령.
2. 버텍스 셰이더: 정점 하나의 위치와 속성 계산.
3. 프래그먼트 셰이더: 프래그먼트 하나의 색 계산.

### 버텍스 셰이더: 위치와 색 전달

```glsl
#version 300 es
layout(location = 0) in vec3 aPos;   // 위치 버퍼에서 받은 정점 하나
layout(location = 3) in vec3 aColor; // 색 버퍼에서 받은 색 하나
out vec3 vColor;                    // 다음 단계로 내보낼 값

void main() {
  gl_Position = vec4(aPos, 1.0); // 클립 좌표 출력. 여기서는 w=1
  vColor = aColor;              // 세 꼭짓점의 색을 넘김
}
```

`gl_Position`은 **클립 좌표**입니다. 이후 xyz를 w로 나눠 NDC가 됩니다. 위 예제는 w가 1이어서 xyz가 그대로 NDC가 되며, 보이는 범위는 각 축 -1~1입니다. 삼각형이 경계를 가로지르면 화면 안쪽 부분은 남습니다.

### 프래그먼트 셰이더: 보간된 색 출력

```glsl
#version 300 es
precision highp float; // 프래그먼트 셰이더의 float 기본 정밀도
in vec3 vColor;        // 꼭짓점 사이를 섞은 색이 들어옴
out vec4 fragColor;    // 최종 출력 색

void main() {
  fragColor = vec4(vColor, 1.0); // RGB + 불투명도 1
}
```

꼭짓점에 빨강·초록·파랑만 지정해도 삼각형 내부에는 그라데이션이 생깁니다.

## 5. GLSL — 꼭 필요한 문법과 함수

### 자료형과 연산

```glsl
float time = 1.0;             // 실수는 소수점을 붙여 쓰는 습관
vec2 uv = vec2(0.2, 0.8);     // 실수 2개
vec3 color = vec3(1.0, 0.5, 0.0); // 실수 3개: 주황색
vec4 rgba = vec4(color, 1.0); // vec3 뒤에 성분 하나 추가
vec3 reversed = color.bgr;    // 성분 순서 바꾸기: 스위즐링

vec3 a = vec3(1.0, 2.0, 3.0);
vec3 b = vec3(4.0, 5.0, 6.0);
vec3 product = a * b;         // 성분별 곱: (4, 10, 18)
float inner = dot(a, b);      // 내적: 4 + 10 + 18 = 32
// mat4는 4×4 행렬. M * vec4(position, 1.0)으로 위치를 변환합니다.
```

### in, out, uniform

| 선언 위치 | 의미 |
|---|---|
| 버텍스의 `in` | 정점 버퍼에서 읽는 속성 |
| 버텍스의 `out` | 래스터화 단계로 넘기는 값 |
| 프래그먼트의 `in` | 보간해서 받은 값 |
| 프래그먼트의 `out` | 색 출력 |
| `uniform` | 한 번의 그리기 동안 공통으로 사용하는 값 |

```js
const timeLocation = gl.getUniformLocation(program, 'uTime');
gl.useProgram(program);             // 값을 받을 프로그램부터 선택
gl.uniform1f(timeLocation, 2.0);     // uniform float uTime에 2초 전달
```

uniform은 프레임마다, 물체를 그릴 때마다 바꿀 수 있습니다. 다른 프로그램의 위치를 쓰거나 프로그램 선택 없이 값을 설정하면 WebGL 오류가 발생할 수 있으므로 콘솔도 확인하세요.

### 무늬 만들기에 쓰는 함수

```glsl
float x = 0.6;
float a = step(0.5, x);            // x<0.5면 0, 아니면 1 → 여기서는 1
float b = smoothstep(0.3, 0.7, x); // 0.3~0.7 구간에서 부드럽게 0→1
float c = fract(2.7);              // x-floor(x): 약 0.7, 반복 무늬에 사용
float d = clamp(1.4, 0.0, 1.0);    // 0~1 범위로 제한 → 1
vec3 sea = vec3(0.0, 0.2, 0.6);
vec3 land = vec3(0.2, 0.6, 0.1);
vec3 coast = mix(sea, land, b);     // b가 0이면 바다, 1이면 육지
```

`mix(a,b,t) = a*(1-t) + b*t`입니다. `smoothstep`의 두 경계 사이가 넓으면 전환이 더 넓게 번집니다. **경계는 작은 값부터 넣으세요.** 원문의 일부 예제처럼 경계를 역순으로 넣는 방식은 GLSL에서 결과가 정의되지 않습니다. 반대로 만들려면 `1.0 - smoothstep(작은값, 큰값, x)`를 사용합니다.

| 함수 | 쓰임 |
|---|---|
| `abs`, `floor`, `fract`, `mod` | 대칭, 격자, 반복 |
| `min`, `max`, `clamp` | 값 제한 |
| `normalize`, `dot`, `cross` | 단위 방향, 내적, 외적 |
| `length`, `distance` | 길이와 거리 |
| `sin`, `cos`, `atan` | 파동과 각도; 각도 단위는 라디안 |
| `pow`, `sqrt`, `exp`, `log` | 강조 곡선과 수학 계산 |
| `reflect` | 입사 방향의 반사 방향 |
| `dFdx`, `dFdy` | 프래그먼트 단계에서 화면 방향의 변화율 계산 |

`smoothstep`으로 경계를 부드럽게 해도 모든 계단 현상이 자동으로 없어지는 것은 아닙니다. 무늬가 너무 촘촘하면 화면 해상도에 맞춘 추가 처리가 필요합니다.

## 6. 보간 — 꼭짓점 사이의 값은 GPU가 채운다

세 꼭짓점의 값을 비율에 맞게 섞으면 내부 값이 됩니다. w가 모두 같은 단순한 경우 다음처럼 이해할 수 있습니다.

```glsl
// 설명용 계산입니다. 실제 보간은 GPU가 자동으로 합니다.
vec3 red   = vec3(1.0, 0.0, 0.0);
vec3 green = vec3(0.0, 1.0, 0.0);
vec3 blue  = vec3(0.0, 0.0, 1.0);
vec3 center = (red + green + blue) / 3.0; // 중심은 세 색이 같은 비율
```

원근 투영에서는 기본적으로 원근 보정 보간을 사용합니다. 색뿐 아니라 UV와 법선도 보간됩니다. 서로 다른 단위 법선을 섞으면 길이가 보통 1보다 작아지므로 **프래그먼트에서 다시 `normalize`**합니다. 보간을 끄려면 양쪽 선언에 `flat`을 붙이며, 정수 varying은 `flat`을 사용합니다.

## 7. 확산 반사 — 표면 방향으로 밝기를 정한다

면이 빛을 정면으로 받으면 밝고 비스듬히 받으면 어둡습니다. 같은 빛이 더 넓은 면적에 퍼지기 때문입니다. 밝기는 법선과 빛 방향 사이 각도의 코사인에 비례합니다.

```glsl
vec3 N = normalize(vNormal);   // 바깥쪽 표면 방향, 길이 1
vec3 L = normalize(uLightDir); // 표면에서 광원 쪽을 향하는 방향, 길이 1
float diff = max(dot(N, L), 0.0); // 내적=cosθ, 뒷면의 음수는 0으로
vec3 lit = vColor * (0.15 + 0.85 * diff); // 최소 밝기 + 확산광
fragColor = vec4(lit, 1.0);
```

- 같은 방향: `dot=1`, 가장 밝음.
- 수직: `dot=0`, 확산광 없음.
- 반대 방향: `dot<0`, `max`로 0 처리.

위의 `0.15`는 어두운 면도 보이게 하는 간단한 주변광 흉내입니다. 실제 주변광이나 그림자를 계산한 것은 아닙니다. 확산 반사에는 카메라 방향이 필요하지 않으며, 반짝임은 별도의 계산입니다. N과 L은 반드시 같은 좌표계에서 계산하세요.

## 8. 셰이더 실습실 — 한 가지씩 바꿔 보기

버텍스 셰이더를 바꾸면 모양과 위치가, 프래그먼트 셰이더를 바꾸면 색과 무늬가 달라집니다.

```glsl
// 버텍스 main() 안: 정점 위치를 시간에 따라 좌우로 이동
vec3 p = aPos;
p.x += sin(uTime) * 0.15; // 범위 -0.15~0.15
gl_Position = vec4(p, 1.0);
```

```glsl
// 프래그먼트 main() 안: 정점에서 넘긴 물체 좌표 vPos로 줄무늬 생성
float stripe = 0.5 + 0.5 * sin(vPos.y * 20.0 + uTime);
// sin의 -1~1을 0~1로 바꿔 두 색의 혼합 비율로 사용
vec3 color = mix(vec3(0.1, 0.2, 0.8), vec3(1.0, 0.8, 0.2), stripe);
fragColor = vec4(color, 1.0);
```

`20.0`을 늘리면 띠가 촘촘해지고, 시간 항을 지우면 정지합니다. `sin(y*20)`은 띠가 정확히 20개라는 뜻이 아닙니다. 반복 수는 y의 범위와 주기 2π에 따라 달라집니다.

## 9. 실습 — 삼각형에서 공, 그리고 행성으로

### 실습 1: 바닥 위에 떠서 도는 공

강의의 단계는 다음과 연결됩니다.

1. 출발점 파일에서 메시·셰이더·버퍼·프레임 함수의 위치를 찾습니다.
2. 깊이 테스트를 켜서 앞의 면이 뒤의 면을 가리게 합니다.
3. 바닥용 상자와 공용 구 메시를 만듭니다.
4. 셰이더에 모델 행렬과 뷰·투영 행렬을 전달합니다.
5. 이동·회전·크기 행렬 도구와 카메라 변환을 구성합니다.
6. 바닥과 공을 각자의 모델 행렬로 그립니다.
7. 프래그먼트 셰이더에 확산광을 적용합니다.
8. 실습실에서 만든 변형과 무늬를 하나씩 가져옵니다.

```js
gl.enable(gl.DEPTH_TEST); // 보통 초기화할 때 한 번 설정
// 프레임 시작마다 이전 색과 깊이를 모두 지웁니다.
gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);
```

```glsl
// 버텍스 셰이더의 변환 핵심. 행렬과 입출력 선언은 별도로 필요합니다.
gl_Position = uViewProj * uModel * vec4(aPos, 1.0);
// 오른쪽부터 적용: 물체→세계→카메라·투영
vNormal = mat3(uModel) * aNormal;
// 위 법선식은 회전+균일 크기 변환에서 정규화와 함께 사용합니다.
// 비균일 크기 변환에는 모델 행렬 3×3 부분의 역전치가 필요합니다.
```

행렬 곱은 순서가 중요합니다. `T * R * S * p`라면 크기 조절 → 회전 → 이동입니다. 카메라 행렬은 카메라의 위치 자체가 아니라 세계를 카메라 기준으로 바꾸는 변환이라는 점도 기억하세요.

### 실습 2: 이미지 없이 살아 있는 행성 만들기

핵심은 이미지 파일 대신 **좌표를 함수에 넣어 표면 색을 계산**하는 것입니다. 원문 예시는 가스 행성(띠와 폭풍), 바위 행성(대륙과 구름), 위성(분화구와 요철)입니다.

```glsl
// 버텍스 main() 안: 두 종류의 방향을 나눠 보냅니다.
vNormal = mat3(uModel) * aNormal; // 세계 기준: 고정된 빛과 비교할 방향
vSurf = aNormal;                 // 물체 기준: 자전 전의 표면 방향
// vSurf를 사용하면 무늬가 회전하는 물체에 붙어 있게 됩니다.
```

```glsl
// 프래그먼트 main() 안: 구 표면에 붙는 띠와 극지방 무늬
vec3 S = normalize(vSurf); // 원점 중심 구의 표면 방향
float lat = S.y;          // -1~1: 위도의 대용값, 실제 위도 각도는 아님
float lon = atan(S.z, S.x); // -π~π: 경도
float band = 0.5 + 0.5 * sin(lat * 18.0 + sin(lon * 3.0 + uTime));
vec3 base = mix(vec3(0.15, 0.3, 0.6), vec3(0.8, 0.6, 0.3), band);
float ice = smoothstep(0.65, 0.85, abs(lat)); // 남·북극으로 갈수록 1
base = mix(base, vec3(0.95), ice); // 극지방을 흰색으로 덮음
vec3 N = normalize(vNormal);
float diff = max(dot(N, normalize(uLightDir)), 0.0);
fragColor = vec4(base * (0.08 + 0.92 * diff), 1.0);
```

잡음은 세 단계로 이해하면 됩니다.

| 단계 | 역할 |
|---|---|
| hash | 같은 좌표에 항상 같은 가짜 난수 부여 |
| value noise | 격자 꼭짓점 값들을 부드럽게 보간 |
| fBm | 좌표 주파수는 높이고 진폭은 줄이며 잡음을 여러 겹 합산 |

아래는 fBm의 구조만 보여 주는 조각입니다. `noise3` 함수의 구현은 별도로 필요합니다.

```glsl
float fbm(vec3 p) {
  float sum = 0.0;
  float amplitude = 0.5;
  for (int i = 0; i < 5; i++) {
    sum += amplitude * noise3(p); // 큰 얼룩부터 작은 얼룩까지 합산
    p *= 2.0;                    // 공간 주파수 2배: 무늬 크기는 절반
    amplitude *= 0.5;            // 작은 무늬의 영향은 줄임
  }
  return sum;
}
```

분화구는 `floor`로 칸을 구분하고, 칸마다 중심과 반지름을 정한 뒤 `distance`로 둥근 마스크를 만들 수 있습니다. 높이 변화로 법선을 조절하면 메시를 바꾸지 않고 요철을 흉내 낼 수 있지만, 화면 미분을 세계 법선에 단순히 더하는 방식은 정확한 범프 매핑이 아닙니다.

과제에서는 서로 다른 행성 두 개 이상, `uTime`에 의한 움직임, 낮과 밤의 경계, 예시와 구분되는 의도를 요구합니다. 보고서에는 만들려던 모습, 두 가지 이상의 무늬 계산식과 이유, 수정 과정, 캡처와 프래그먼트 셰이더 코드를 정리합니다. 제출 파일명은 원문 기준 `week3.md`, `task1.html`, `task2.html`이며 GitHub Pages 링크도 넣습니다.

## 10. 통째로 실행하는 최소 예제

아래 코드를 `triangle-study.html`로 저장하고 브라우저로 열면 됩니다. 외부 라이브러리 없이 색이 보간되는 삼각형이 좌우로 움직입니다. 완성된 과제용 3D 장면이 아니라 **버퍼 → 버텍스 → 보간 → 프래그먼트** 흐름을 보는 예제입니다.

```html
<!doctype html>
<html lang="ko">
<meta charset="utf-8">
<title>삼각형으로 배우는 WebGL2</title>
<style>
  body { margin: 0; background: #111827; color: white; text-align: center; }
  canvas { width: min(90vw, 600px); height: auto; }
</style>
<p>정점의 세 가지 색이 안쪽에서 섞입니다. 소스의 주석을 함께 읽어 보세요.</p>
<canvas id="canvas" width="600" height="600"></canvas>
<p id="error" role="alert"></p>
<script>
'use strict';
try {
  const canvas = document.getElementById('canvas');
  const gl = canvas.getContext('webgl2');
  if (!gl) throw new Error('WebGL2를 지원하는 브라우저가 필요합니다.');

  // 문자열 맨 처음에 #version을 둡니다. GPU에서 실행할 코드입니다.
  const vs = `#version 300 es
  layout(location=0) in vec3 aPos;   // 정점마다 다른 위치
  layout(location=3) in vec3 aColor; // 정점마다 다른 색
  uniform float uTime;              // 이번 그리기의 공통 시간
  out vec3 vColor;                  // GPU가 보간할 값
  void main() {
    vec3 p = aPos;
    p.x += sin(uTime) * 0.15;       // 모든 정점에 같은 이동량 적용
    gl_Position = vec4(p, 1.0);     // w=1이므로 xyz가 그대로 NDC
    vColor = aColor;
  }`;

  const fs = `#version 300 es
  precision highp float;
  in vec3 vColor;       // 보간된 색. 꼭짓점 색과 다를 수 있음
  out vec4 fragColor;
  void main() {
    fragColor = vec4(vColor, 1.0); // 알파 1: 불투명
  }`;

  // 1. GLSL 문자열을 GPU가 실행할 셰이더로 컴파일합니다.
  function compile(type, source) {
    const shader = gl.createShader(type);
    gl.shaderSource(shader, source);
    gl.compileShader(shader);
    if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
      throw new Error(gl.getShaderInfoLog(shader));
    }
    return shader;
  }
  const program = gl.createProgram();
  gl.attachShader(program, compile(gl.VERTEX_SHADER, vs));
  gl.attachShader(program, compile(gl.FRAGMENT_SHADER, fs));
  gl.linkProgram(program); // 2. 두 셰이더의 입출력을 연결합니다.
  if (!gl.getProgramParameter(program, gl.LINK_STATUS)) {
    throw new Error(gl.getProgramInfoLog(program));
  }

  // 3. VAO는 정점 속성의 읽기 설정과 인덱스 버퍼 연결을 기억합니다.
  const vao = gl.createVertexArray();
  gl.bindVertexArray(vao);
  function attribute(location, values) {
    const buffer = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, buffer);
    gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(values), gl.STATIC_DRAW);
    gl.enableVertexAttribArray(location);
    // 실수 3개씩 읽기, 정규화 없음, 간격·시작 오프셋 0
    gl.vertexAttribPointer(location, 3, gl.FLOAT, false, 0, 0);
  }
  attribute(0, [-0.6,-0.5,0,  0.6,-0.5,0,  0,0.7,0]); // 위치 3개
  attribute(3, [1,0,0,  0,1,0,  0,0,1]);              // 색 3개

  const indexBuffer = gl.createBuffer();
  gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, indexBuffer);
  gl.bufferData(gl.ELEMENT_ARRAY_BUFFER, new Uint16Array([0,1,2]), gl.STATIC_DRAW);
  const timeLocation = gl.getUniformLocation(program, 'uTime');

  // 4. CPU는 시간과 그리기 명령을 전달하고 GPU가 화면을 계산합니다.
  function frame(milliseconds) {
    gl.viewport(0, 0, canvas.width, canvas.height);
    gl.clearColor(0.03, 0.05, 0.1, 1);
    gl.clear(gl.COLOR_BUFFER_BIT); // 삼각형 하나라 깊이 테스트는 생략
    gl.useProgram(program);       // uniform을 넣기 전에 선택
    gl.uniform1f(timeLocation, milliseconds / 1000); // 밀리초→초
    gl.bindVertexArray(vao);
    // 인덱스 3개로 삼각형 하나. Uint16Array와 UNSIGNED_SHORT를 맞춤
    gl.drawElements(gl.TRIANGLES, 3, gl.UNSIGNED_SHORT, 0);
    requestAnimationFrame(frame); // 다음 화면 갱신 때 다시 호출
  }
  requestAnimationFrame(frame);
} catch (error) {
  document.getElementById('error').textContent = error.message;
  console.error(error); // 셰이더 오류의 줄 번호도 개발자 도구에서 확인
}
</script>
</html>
```

예제를 읽고 바꿔 볼 순서:

1. `p.x += ...`를 주석 처리해 움직임을 멈춰 봅니다.
2. 색 배열을 모두 `[1,0,0]`으로 바꿔 단색 삼각형을 만듭니다.
3. `fragColor = vec4(1.0, 0.8, 0.0, 1.0);`으로 바꿔 보간색을 덮어 봅니다.
4. 이동량 `0.15`를 바꾸고 화면 밖으로 나가는 부분이 잘리는지 관찰합니다.

## 11. 막힐 때 확인할 것

| 증상 | 먼저 확인할 부분 |
|---|---|
| 검은 화면 | WebGL2 지원, 컴파일·링크 오류, 좌표 범위, 그리기 호출 |
| 모델이 누움 | y-up과 z-up 혼용 여부 |
| 뒤쪽 면이 앞쪽을 덮음 | 깊이 테스트 활성화와 깊이 버퍼 초기화 |
| 조명이 잘못됨 | 법선 정규화, N과 L의 좌표계, 광원 방향 부호 |
| uniform이 반영되지 않음 | 프로그램 선택, 이름, 위치, uniform 함수 자료형 |
| 셰이더 문법 오류 | GLSL ES 3.00, float 표기, in/out 선언 일치 |
| 무늬가 물체에 붙지 않음 | 물체 기준 좌표와 세계 기준 좌표 구분 |

다음 주에는 모델·뷰·투영 행렬을 체계적으로 다룹니다. 이번 주에는 **숫자 배열을 보내고, 정점을 변환하고, 사이 값을 보간하고, 색을 계산한다**는 연결을 먼저 이해하면 됩니다.
