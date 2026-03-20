# 📏 직사각형 넓이 구하기

## 📋 문제 설명

2차원 좌표 평면에 **변이 축과 평행한 직사각형**이 있습니다.

직사각형 네 꼭짓점의 좌표 `[[x1,y1],[x2,y2],[x3,y3],[x4,y4]]`가 담겨있는 배열 `dots`가 매개변수로 주어질 때, **직사각형의 넓이**를 return하도록 solution 함수를 완성하세요.

---

## 📌 제한 사항

- `dots`의 길이 = 4
- `dots[i]`의 길이 = 2
- -256 ≤ `dots[i][0]`, `dots[i][1]` ≤ 256
- 잘못된 입력은 주어지지 않음 (항상 올바른 직사각형)

---

## 💡 핵심 아이디어

변이 **축과 평행**하므로 꼭짓점 좌표에서 바로 가로/세로 길이를 구할 수 있습니다.

```
가로 길이 = x좌표 최댓값 - x좌표 최솟값
세로 길이 = y좌표 최댓값 - y좌표 최솟값
넓이 = 가로 × 세로
```

---

## ✅ 정답 코드

```cpp
#include <string>
#include <vector>
#include <algorithm>
using namespace std;

int solution(vector<vector<int>> dots) {
    int answer = 0;

    int minX = dots[0][0], maxX = dots[0][0];
    int minY = dots[0][1], maxY = dots[0][1];

    for (int i = 1; i < dots.size(); i++) {
        minX = min(minX, dots[i][0]);
        maxX = max(maxX, dots[i][0]);
        minY = min(minY, dots[i][1]);
        maxY = max(maxY, dots[i][1]);
    }

    answer = (maxX - minX) * (maxY - minY);

    return answer;
}
```

---

## 🔍 코드 분석

### 초기화

```cpp
// 첫 번째 꼭짓점 좌표로 min/max 초기화
int minX = dots[0][0], maxX = dots[0][0];
int minY = dots[0][1], maxY = dots[0][1];
```

### 순회 및 갱신

```cpp
for (int i = 1; i < dots.size(); i++) {
    minX = min(minX, dots[i][0]);  // x 최솟값 갱신
    maxX = max(maxX, dots[i][0]);  // x 최댓값 갱신
    minY = min(minY, dots[i][1]);  // y 최솟값 갱신
    maxY = max(maxY, dots[i][1]);  // y 최댓값 갱신
}
```

| 변수 | 역할 |
|------|------|
| `minX` | x좌표 중 가장 작은 값 (왼쪽 경계) |
| `maxX` | x좌표 중 가장 큰 값 (오른쪽 경계) |
| `minY` | y좌표 중 가장 작은 값 (아래쪽 경계) |
| `maxY` | y좌표 중 가장 큰 값 (위쪽 경계) |

---

## 📐 시각화

```
예시: [[1,1],[4,1],[4,4],[1,4]]

  y
4 ●-----------●  ← (1,4)     (4,4)
  |           |
  |           |  세로 = maxY - minY = 4 - 1 = 3
  |           |
1 ●-----------●  ← (1,1)     (4,1)
  1           4   x
      가로 = maxX - minX = 4 - 1 = 3

  넓이 = 3 × 3 = 9
```

---

## 🧪 테스트 케이스

| dots | 가로 | 세로 | 결과 |
|------|:---:|:---:|:----:|
| `[[1,1],[4,1],[4,4],[1,4]]` | 3 | 3 | **9** |
| `[[0,0],[5,0],[5,3],[0,3]]` | 5 | 3 | **15** |
| `[[2,3],[6,3],[6,7],[2,7]]` | 4 | 4 | **16** |
| `[-1,-1],[3,-1],[3,2],[-1,2]]` | 4 | 3 | **12** |

---

## 🔄 다른 풀이 방법

### 정렬을 이용한 방법

```cpp
int solution(vector<vector<int>> dots) {
    vector<int> x, y;

    for (auto& dot : dots) {
        x.push_back(dot[0]);
        y.push_back(dot[1]);
    }

    sort(x.begin(), x.end());
    sort(y.begin(), y.end());

    return (x.back() - x.front()) * (y.back() - y.front());
}
```

| 방법 | 장점 | 단점 |
|------|------|------|
| min/max 순회 | 추가 배열 불필요, O(N) | 코드가 약간 길다 |
| 정렬 방법 | 코드가 간결 | O(N log N), 추가 배열 필요 |

---

## 📊 시간/공간 복잡도

| 항목 | 복잡도 |
|------|--------|
| 시간 복잡도 | O(N) |
| 공간 복잡도 | O(1) |

- N = 꼭짓점 수 (항상 4로 고정 → 사실상 O(1))

---

## 🏷️ 관련 태그

`#좌표계산` `#최솟값최댓값` `#직사각형` `#넓이`
