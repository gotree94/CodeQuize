# 📦 배열 원소 추가/제거

## 📋 문제 설명

아무 원소도 들어있지 않은 **빈 배열 X**가 있습니다.

길이가 같은 정수 배열 `arr`과 boolean 배열 `flag`가 매개변수로 주어질 때,  
`flag`를 차례대로 순회하며:

- `flag[i]`가 **true** → X의 뒤에 `arr[i]`를 **`arr[i] × 2`번** 추가
- `flag[i]`가 **false** → X에서 **마지막 `arr[i]`개**의 원소를 제거

최종 X를 return하는 solution 함수를 작성하세요.

---

## 📌 제한 사항

- 1 ≤ `arr`의 길이 ≤ 100
- `arr`의 길이 = `flag`의 길이
- 1 ≤ `arr[i]` ≤ 100
- `flag[i]`가 false인 경우 항상 제거 가능한 원소 수가 충분히 존재

---

## 💡 핵심 아이디어

| flag 값 | 동작 | 사용 함수 |
|---------|------|-----------|
| `true` | `arr[i]` 값을 `arr[i] × 2`번 push | `push_back()` 반복 |
| `false` | 마지막 `arr[i]`개 제거 | `pop_back()` 반복 |

> ⚠️ `pop_back()`은 한 번에 원소 **1개만** 제거  
> → `arr[i]`번 반복 호출 필요

---

## ✅ 정답 코드

```cpp
#include <string>
#include <vector>
using namespace std;

vector<int> solution(vector<int> arr, vector<bool> flag) {
    vector<int> answer;

    for (int i = 0; i < arr.size(); i++) {
        if (flag[i]) {
            // arr[i] 값을 arr[i]*2 번 추가
            for (int j = 0; j < arr[i] * 2; j++) {
                answer.push_back(arr[i]);
            }
        } else {
            // 마지막 arr[i]개 제거
            for (int j = 0; j < arr[i]; j++) {
                answer.pop_back();
            }
        }
    }

    return answer;
}
```

---

## 🔍 코드 분석

### 전체 흐름

```
arr  = [3, 2, 4, 1]
flag = [T, F, T, F]

외부 루프: i = 0 ~ arr.size()-1
  ├─ flag[i] == true  → push_back(arr[i])을 arr[i]*2 번 반복
  └─ flag[i] == false → pop_back()을 arr[i] 번 반복
```

### 주요 함수

| 함수 | 기능 |
|------|------|
| `push_back(val)` | 벡터 맨 뒤에 val 삽입 |
| `pop_back()` | 벡터 맨 뒤 원소 1개 제거 |

---

## 🎬 단계별 동작 추적

```
arr  = [3, 2, 4, 1]
flag = [T, F, T, F]

초기 상태: answer = []

─────────────────────────────────────────────────
Step 1: i=0, arr[0]=3, flag[0]=true
  → 3을 3×2=6번 push_back
  answer = [3, 3, 3, 3, 3, 3]

─────────────────────────────────────────────────
Step 2: i=1, arr[1]=2, flag[1]=false
  → 마지막 2개 pop_back
  answer = [3, 3, 3, 3]

─────────────────────────────────────────────────
Step 3: i=2, arr[2]=4, flag[2]=true
  → 4를 4×2=8번 push_back
  answer = [3, 3, 3, 3, 4, 4, 4, 4, 4, 4, 4, 4]

─────────────────────────────────────────────────
Step 4: i=3, arr[3]=1, flag[3]=false
  → 마지막 1개 pop_back
  answer = [3, 3, 3, 3, 4, 4, 4, 4, 4, 4, 4]

─────────────────────────────────────────────────
최종 반환: [3, 3, 3, 3, 4, 4, 4, 4, 4, 4, 4]
```

---

## 🧪 테스트 케이스

### 예시 1
```
arr  = [3, 2, 4, 1]
flag = [true, false, true, false]

결과: [3, 3, 3, 3, 4, 4, 4, 4, 4, 4, 4]
```

### 예시 2
```
arr  = [1, 2]
flag = [true, true]

Step 1: 1을 1×2=2번 추가 → [1, 1]
Step 2: 2를 2×2=4번 추가 → [1, 1, 2, 2, 2, 2]

결과: [1, 1, 2, 2, 2, 2]
```

### 예시 3
```
arr  = [5, 3]
flag = [true, false]

Step 1: 5를 5×2=10번 추가 → [5,5,5,5,5,5,5,5,5,5]
Step 2: 마지막 3개 제거    → [5,5,5,5,5,5,5]

결과: [5, 5, 5, 5, 5, 5, 5]
```

---

## ⚠️ 자주 하는 실수

```cpp
// ❌ 잘못된 예 - arr[i]를 1번만 push (arr[i]*2 번이 아님)
for (int j = 0; j < arr[i]; j++) {
    answer.push_back(arr[i]);
}

// ✅ 올바른 예 - arr[i]*2 번 push
for (int j = 0; j < arr[i] * 2; j++) {
    answer.push_back(arr[i]);
}
```

---

## 📊 시간/공간 복잡도

| 항목 | 복잡도 |
|------|--------|
| 시간 복잡도 | O(N × M) |
| 공간 복잡도 | O(N × M) |

- N: arr의 길이, M: arr[i]의 최댓값
- 최대 원소 삽입 횟수 = arr[i] × 2 의 누적 합

---

## 🏷️ 관련 태그

`#vector` `#push_back` `#pop_back` `#배열조작` `#스택`
