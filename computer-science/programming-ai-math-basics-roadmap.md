# 프로그래밍과 AI를 위한 수학 기초 — 개발자 수학 로드맵

수학은 AI와 알고리즘의 언어임. 겁먹을 필요 없고, 개발자 관점에서 '왜 쓰이는가'를 먼저 잡으면 생각보다 빠르게 익힐 수 있음. 이 글은 처음 수학을 공부하는 개발자를 위한 빌드업 로드맵임.

---

## 1. 왜 개발자에게 수학이 필요한가

수학을 몰라도 코드를 짤 수 있음. 하지만 다음 상황에서 반드시 막히게 됨.

**알고리즘 설계**: 시간 복잡도 O(n log n)이 왜 O(n²)보다 빠른지 이해하려면 로그(log) 개념이 필요함.

**머신러닝 / 딥러닝**: 모델이 학습되는 원리(경사하강법), 예측 오류를 줄이는 방법(손실 함수), 데이터를 표현하는 방법(벡터·행렬) 모두 수학임.

**그래픽스 / 게임 엔진**: 회전·이동·투영 변환은 행렬 곱셈으로 구현됨.

**암호학 / 보안**: RSA 암호화는 소수론과 모듈러 산술 위에 세워짐.

**데이터 분석**: 평균, 분산, 상관관계, 정규분포를 모르면 데이터를 해석할 수 없음.

> 수학을 공부하는 목적은 수식을 외우는 게 아니라, 개념의 구조를 이해해서 코드로 구현하거나 논문을 읽을 수 있는 역량을 키우는 것임.

---

## 2. 수학 표기법 읽는 법

논문이나 교재를 보면 기호가 가득함. 먼저 기호에 익숙해지는 것이 첫 번째 관문.

### 2-1. 자주 쓰는 기호

| 기호 | 영어 읽기 | 의미 | 예시 |
|---|---|---|---|
| `Σ` | Sigma | 합산 (for loop의 수학 버전) | `Σᵢ₌₁ⁿ i = 1+2+…+n` |
| `Π` | Pi | 곱셈 | `Πᵢ₌₁ⁿ i = n!` |
| `∀` | For all | 모든 ~ 에 대해 | `∀x ∈ ℝ, x² ≥ 0` |
| `∃` | There exists | ~가 존재한다 | `∃x : x² = 4` |
| `∈` | Element of | ~의 원소 | `3 ∈ {1,2,3}` |
| `⊆` | Subset of | 부분집합 | `{1,2} ⊆ {1,2,3}` |
| `→` | Maps to | 함수 정의역 → 치역 | `f: ℝ → ℝ` |
| `∇` | Nabla / del | 그래디언트 (편미분 벡터) | `∇f` |
| `‖v‖` | Norm | 벡터의 크기(길이) | `‖[3,4]‖ = 5` |
| `∞` | Infinity | 무한대 | `limₓ→∞ 1/x = 0` |

### 2-2. Σ (시그마) — 개발자 관점

```python
# 수학: Σᵢ₌₁ⁿ i
# 코드로 읽으면 그냥 for loop

def sigma(n):
    total = 0
    for i in range(1, n + 1):
        total += i
    return total

# 또는
result = sum(range(1, n + 1))

# Σᵢ₌₁ⁿ xᵢ² → 각 원소를 제곱해서 다 더하기
x = [1, 2, 3, 4, 5]
result = sum(xi**2 for xi in x)  # = 1+4+9+16+25 = 55
```

### 2-3. 함수 표기법

```
f: A → B
```

- `A` = 정의역 (Domain): 입력값의 집합
- `B` = 공역 (Codomain): 출력값이 올 수 있는 집합
- `f(A)` = 치역 (Range/Image): 실제 출력값의 집합 (공역의 부분집합)

```python
# f: ℤ → ℤ,  f(x) = x²
# 정의역: 정수 전체 / 공역: 정수 전체 / 치역: 0, 1, 4, 9, 16, ...
def f(x: int) -> int:
    return x ** 2
```

---

## 3. 집합론 기초

집합(Set)은 수학 전체의 언어임. 데이터베이스의 테이블, 프로그래밍의 타입 시스템, 확률론 모두 집합 위에 세워져 있음.

### 3-1. 집합 연산

```python
A = {1, 2, 3, 4}
B = {3, 4, 5, 6}

union     = A | B    # 합집합  A ∪ B → {1, 2, 3, 4, 5, 6}
intersect = A & B    # 교집합  A ∩ B → {3, 4}
diff      = A - B    # 차집합  A \ B → {1, 2}
sym_diff  = A ^ B    # 대칭차  A △ B → {1, 2, 5, 6}

print({1, 2} <= A)   # True  (⊆ 부분집합)
print({1, 2} < A)    # True  (⊊ 진부분집합)
```

### 3-2. 실무 연결 — SQL의 집합 연산

```sql
-- UNION     : A ∪ B
SELECT id FROM table_a UNION SELECT id FROM table_b;

-- INTERSECT : A ∩ B
SELECT id FROM table_a INTERSECT SELECT id FROM table_b;

-- EXCEPT    : A \ B
SELECT id FROM table_a EXCEPT SELECT id FROM table_b;
```

---

## 4. 함수의 성질 — 단사·전사·전단사

AI와 암호학에서 핵심적으로 쓰이는 개념.

```python
# 단사 (Injective): 서로 다른 입력 → 반드시 다른 출력
# f(x) = 2x  →  f(1)=2, f(2)=4 (겹치지 않음)
# 역함수 존재 조건, 암호화 함수의 필수 성질

def is_injective(func, domain):
    outputs = [func(x) for x in domain]
    return len(outputs) == len(set(outputs))

# 전사 (Surjective): 공역의 모든 원소가 최소 1번 커버됨
# 해시 함수: 무한 입력 → 유한 출력 → 전사O 단사X (충돌 발생)

# 전단사 (Bijective): 단사 + 전사 = 완전한 1:1 대응
# f(x) = x+1  →  f⁻¹(y) = y-1 (역함수 유일)
# RSA, AES 암호화의 수학적 기반
```

---

## 5. 선형대수 입문 — 벡터와 행렬

AI의 핵심 도구. 신경망의 모든 연산은 행렬 곱셈으로 표현됨.

### 5-1. 벡터 (Vector)

```python
import numpy as np

# 벡터: 방향과 크기를 가진 수의 배열
# 실무: 데이터 포인트 하나 = 벡터  (예: [나이=25, 구매횟수=3, 평점=4.5])

v = np.array([3, 4])
u = np.array([1, 2])

print(v + u)                   # [4, 6]  — 벡터 덧셈
print(2 * v)                   # [6, 8]  — 스칼라 곱
print(np.linalg.norm(v))       # 5.0     — 노름 ||v|| = √(3²+4²)
print(np.dot(v, u))            # 11      — 내적 (유사도 측정)

# 코사인 유사도: 두 벡터의 방향이 얼마나 같은지 (추천 시스템)
cos_sim = np.dot(v, u) / (np.linalg.norm(v) * np.linalg.norm(u))
print(cos_sim)  # 0.9838...  (1에 가까울수록 유사)
```

### 5-2. 행렬 (Matrix)

```python
import numpy as np

A = np.array([[1, 2], [3, 4]])
B = np.array([[5, 6], [7, 8]])

print(A + B)    # 성분별 덧셈
print(A @ B)    # 행렬 곱셈  [[19,22],[43,50]]
print(A.T)      # 전치행렬   [[1,3],[2,4]]

# 신경망 순전파: z = xW + b
x = np.random.randn(4, 3)   # 입력 (배치4, 특성3)
W = np.random.randn(3, 5)   # 가중치 (특성3 → 은닉5)
b = np.zeros(5)
z = x @ W + b               # (4, 5) 출력
```

### 5-3. 연립방정식 풀기

```python
import numpy as np

# 2x + y = 4
# 5x + 3y = 7
A = np.array([[2, 1], [5, 3]])
b = np.array([4, 7])

x = np.linalg.solve(A, b)
print(x)  # [5. -6.]  → x=5, y=-6
```

---

## 6. 확률과 통계 기초

AI 모델의 출력은 '확률'임. 확률을 모르면 모델 결과를 해석할 수 없음.

### 6-1. 핵심 통계량

```python
import numpy as np

data = np.array([2, 4, 4, 4, 5, 5, 7, 9])

mean   = np.mean(data)    # 5.0  — 평균
var    = np.var(data)     # 4.0  — 분산 Var(X) = E[(X-μ)²]
std    = np.std(data)     # 2.0  — 표준편차
median = np.median(data)  # 4.5  — 중앙값 (이상치에 강건)

# Min-Max 정규화: 0~1 범위로 조정
normalized   = (data - data.min()) / (data.max() - data.min())

# Z-score 표준화: 평균=0, 표준편차=1
standardized = (data - mean) / std
```

### 6-2. 확률분포와 소프트맥스

```python
import numpy as np

# 정규분포: 딥러닝 가중치 초기화에 사용
weights = np.random.normal(loc=0, scale=0.01, size=(100, 100))

# 소프트맥스: 다중 클래스 확률 변환 (분류 모델 출력층)
def softmax(x):
    exp_x = np.exp(x - np.max(x))   # 오버플로 방지
    return exp_x / exp_x.sum()

logits = np.array([2.0, 1.0, 0.1])
probs  = softmax(logits)
print(probs)        # [0.659, 0.242, 0.099]
print(probs.sum())  # 1.0
```

---

## 7. 미적분 핵심 — 경사하강법의 수학

딥러닝 학습의 본질은 미분임.

### 7-1. 편미분과 그래디언트

```python
import numpy as np

# 편미분: 여러 변수 중 하나만 변화시켰을 때의 변화율
# f(x, y) = x² + 2xy + y²  →  ∂f/∂x = 2x + 2y

def f(x, y):    return x**2 + 2*x*y + y**2
def partial_x(x, y, h=1e-5): return (f(x+h, y) - f(x-h, y)) / (2*h)
def partial_y(x, y, h=1e-5): return (f(x, y+h) - f(x, y-h)) / (2*h)

print(partial_x(2.0, 3.0))  # 10.0
print(partial_y(2.0, 3.0))  # 10.0
```

### 7-2. 경사하강법 직접 구현

```python
# θ ← θ - α × ∇L(θ)
# θ: 파라미터  α: 학습률  L: 손실함수  ∇L: 그래디언트

def f(x):  return x**2 - 4*x + 4   # (x-2)² 최솟값: x=2
def df(x): return 2*x - 4           # 그래디언트

x, alpha = 10.0, 0.1

for step in range(30):
    x = x - alpha * df(x)           # 그래디언트 반대 방향으로 이동
    if step % 5 == 0:
        print(f"step {step:2d}: x = {x:.4f}, f(x) = {f(x):.6f}")

# step  0: x = 8.8000, f(x) = 47.000000
# step  5: x = 4.2634, f(x) = 5.041605
# step 25: x = 2.0013, f(x) = 0.000002
# 최솟값 근사: x ≈ 2.0 (정답: x = 2.0)
```

---

## 8. 수학 학습 로드맵

### 영역별 우선순위

| 수학 영역 | 필요한 분야 | 우선순위 |
|---|---|---|
| 선형대수 (벡터·행렬) | 머신러닝, 딥러닝, 그래픽스 | ⭐⭐⭐ 최우선 |
| 확률·통계 | 머신러닝, 데이터 분석, 베이즈 학습 | ⭐⭐⭐ 최우선 |
| 미적분 (편미분·그래디언트) | 딥러닝 역전파, 최적화 | ⭐⭐⭐ 최우선 |
| 이산 수학 (집합·논리·그래프) | 알고리즘, 자료구조, DB | ⭐⭐ 중요 |
| 정보 이론 (엔트로피·KL 발산) | 딥러닝 손실함수, 압축 | ⭐⭐ 중요 |
| 수치 해석 | 시뮬레이션, 과학 컴퓨팅 | ⭐ 심화 |

### 단계별 학습 순서

```
[1단계] 기초 언어 습득
  수학 기호 읽기 → 집합론 기초 → 함수의 개념
  목표: 수식을 보고 도망가지 않게 되기

[2단계] 선형대수
  벡터와 내적 → 행렬 연산 → 역행렬·행렬식 → 고유값·고유벡터
  목표: NumPy로 직접 구현하며 이해

[3단계] 미적분
  극한·연속 → 도함수 → 편미분 → 그래디언트 → 체인룰
  목표: 경사하강법 직접 구현

[4단계] 확률·통계
  확률 기초 → 조건부 확률·베이즈 → 확률분포 → 기댓값·분산
  목표: 소프트맥스, 손실함수, 정규화 이해

[5단계] 통합
  위 개념을 결합해 간단한 신경망 직접 구현 (NumPy만으로)
  목표: 수학이 코드로 어떻게 연결되는지 체감
```

### 추천 학습 자료

- **선형대수**: 3Blue1Brown "Essence of Linear Algebra" (YouTube, 무료)
- **미적분**: 3Blue1Brown "Essence of Calculus" (YouTube, 무료)
- **확률·통계**: StatQuest with Josh Starmer (YouTube, 무료)
- **종합 교재**: Mathematics for Machine Learning (캠브리지 대학, PDF 무료 공개)
- **실습**: Khan Academy → 개념 → NumPy로 직접 구현

---

## 요약

| 개념 | 한 줄 요약 | 실무 연결 |
|---|---|---|
| Σ / 수열 | for loop의 수학 표현 | 손실 함수 계산 |
| 집합론 | 데이터 분류와 관계의 언어 | SQL 집합 연산, 타입 시스템 |
| 함수 성질 | 입출력 대응 구조 | 암호화, 해시함수 설계 |
| 벡터·내적 | 방향과 유사도 | 추천 시스템, 임베딩 |
| 행렬 곱셈 | 변환과 투영의 도구 | 신경망 레이어, 그래픽스 |
| 편미분·그래디언트 | 어떤 방향으로 바꿔야 최솟값인가 | 역전파, 경사하강법 |
| 확률분포 | 불확실성을 수로 표현 | 분류 모델 출력, 데이터 생성 |

수학 공부의 핵심은 수식 암기가 아니라 **개념이 왜 필요한지, 코드로 어떻게 연결되는지**를 체감하는 것임. 빌드업은 여기서 시작하고, 다음 글에서 각 영역을 더 깊이 파고들 예정.
