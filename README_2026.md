# MFC 대화상자 기반 — 세 점 외접원 프로그램 구현 가이드

> **버전 노트**: 이 문서는 **Visual Studio 2026** 기준으로 작성되었습니다.  
> VS 2026은 Fluent UI 기반으로 전면 재설계되었으며, 설정 시스템 및 일부 메뉴 구조가 VS 2022와 다릅니다.  
> VS 2022와의 차이점은 각 섹션에서 `🔄 VS 2022와의 차이` 항목으로 별도 표기합니다.

## 목차
1. [프로젝트 개요 및 요구사항](#1-프로젝트-개요-및-요구사항)
2. [개발 환경 및 프로젝트 생성](#2-개발-환경-및-프로젝트-생성)
3. [STEP 01 — 빈 대화상자에 마우스 클릭으로 점 찍기](#3-step-01--빈-대화상자에-마우스-클릭으로-점-찍기)
4. [STEP 02 — 3점 완성 시 외접원 계산 및 표시](#4-step-02--3점-완성-시-외접원-계산-및-표시)
5. [STEP 03 — 계산 과정 시각화 (수직이등분선 등)](#5-step-03--계산-과정-시각화-수직이등분선-등)
6. [STEP 04 — 사용자 입력: 점 반지름 / 외접원 두께](#6-step-04--사용자-입력-점-반지름--외접원-두께)
7. [STEP 05 — Ellipse API 금지: Polyline으로 원 그리기](#7-step-05--ellipse-api-금지-polyline으로-원-그리기)
8. [STEP 06 — 점 드래그로 외접원 실시간 재계산](#8-step-06--점-드래그로-외접원-실시간-재계산)
9. [STEP 07 — 초기화 버튼](#9-step-07--초기화-버튼)
10. [STEP 08 — 랜덤 이동 (별도 쓰레드, 프리징 없음)](#10-step-08--랜덤-이동-별도-쓰레드-프리징-없음)
11. [전체 소스 코드](#11-전체-소스-코드)
12. [빌드 및 실행](#12-빌드-및-실행)
13. [동작 요약](#13-동작-요약)
14. [트러블슈팅](#14-트러블슈팅)

---

## 1. 프로젝트 개요 및 요구사항

### 1.1 최종 동작 화면 구성

```
┌─────────────────────────────────────────────────────┬──────────────┐
│                                                     │  [설정]      │
│          캔버스 영역 (700px)                         │  점 원 반지름│
│                                                     │  [   10    ] │
│   P₁●────────────────────P₂●                       │  외접원 두께 │
│      \   수직이등분선L₁  /                          │  [    2    ] │
│       \       |         /                           │              │
│        \      O        /   ← 외접원                 │  [  초기화  ]│
│         \     |       /                             │  [랜덤 이동 ]│
│          P₃●──────────                              │              │
│                                                     │  P1:(x,y)    │
│                                                     │  P2:(x,y)    │
│                                                     │  P3:(x,y)    │
│                                                     │  O:(cx,cy)   │
│                                                     │  r=xxx.x     │
└─────────────────────────────────────────────────────┴──────────────┘
```

### 1.2 전체 요구사항 목록

| # | 요구사항 | 구현 단계 |
|---|----------|-----------|
| R1 | 마우스 좌클릭으로 점 찍기 | STEP 01 |
| R2 | 3번째 클릭 시 외접원 자동 계산 및 표시 | STEP 02 |
| R3 | 계산 과정(중점, 수직이등분선, 반지름선) 시각화 | STEP 03 |
| R4 | 클릭 지점 원 반지름 / 외접원 두께 사용자 입력 | STEP 04 |
| R5 | `Ellipse` 등 고수준 API 금지 → `Polyline` 사용 | STEP 05 |
| R6 | 점 클릭 후 드래그 시 외접원 실시간 재계산 | STEP 06 |
| R7 | 4번째 클릭부터 점 원 미표시 | STEP 06 |
| R8 | [초기화] 버튼 → 전체 초기화 | STEP 07 |
| R9 | [랜덤 이동] → 별도 쓰레드, 초당 2회, 총 10회 | STEP 08 |
| R10 | 메인 UI 프리징 없음 | STEP 08 |

### 1.3 금지 API

```
wingdi, gdiplus, Ellipse류, FillPolygon, DrawPolygon
```
원은 반드시 **삼각함수 + `Polyline`** 으로 직접 그려야 합니다.

---

## 2. 개발 환경 및 프로젝트 생성

### 2.1 개발 환경

| 항목 | 사양 |
|------|------|
| IDE | **Visual Studio 2026** (v18) |
| UI 디자인 | Fluent UI 기반 전면 재설계 |
| 언어 | C++ (MFC) |
| 프로젝트 형식 | MFC 앱 — 대화 상자 기반 |
| 문자 집합 | 유니코드 |
| 권장 설치 채널 | Stable 또는 Insiders (월별 자동 업데이트) |

> VS 2026은 VS 2022와 **나란히 설치(Side-by-side)** 가능합니다.  
> 기존 VS 2022의 테마, 단축키, 확장 설정을 Setup Assistant로 자동 가져올 수 있습니다.

---

### 2.2 MFC 워크로드 설치 확인

VS 2026에서 MFC 프로젝트를 사용하려면 C++ 워크로드가 설치되어 있어야 합니다.

**① Visual Studio Installer 실행**

```
Windows 시작 메뉴 → "Visual Studio Installer" 검색 후 실행
또는 VS 2026 메뉴 → 도구 → 도구 및 기능 가져오기...
```

**② 워크로드 확인**

```
워크로드 탭 → "C++를 사용한 데스크톱 개발" 체크 확인
```

**③ 개별 구성 요소에서 MFC 확인**

검색창에 `MSVC MFC` 로 입력하면 아래와 같이 항목이 분류되어 표시됩니다.

```
📦 SDK, 라이브러리 및 프레임워크
  ──────────────────────────────────────────────────────
  □ ARM64/ARM64EC용 C++ MFC(최신 MSVC)
  □ ARM64/ARM64EC용 Spectre 완화 기능이 적용된 C++ MFC(최신 MSVC)
  □ ARM64용 C++ MFC(MSVC v14.50)
  □ ARM64용 Spectre 완화 기능이 포함된 C++ MFC(MSVC v14.50)
  ✅ x64/x86용 C++ MFC(MSVC v14.50)           ← 일반 PC 개발 필수
  ✅ x64/x86용 C++ MFC(최신 MSVC)             ← 권장 (둘 중 하나 이상)
  □ x64/x86용 Spectre 완화 기능이 적용된 C++ MFC (MSVC v14.50)
  □ x64/x86용 Spectre 완화 기능이 적용된 C++ MFC(최신 MSVC)

🔍 미리 보기 SDK, 라이브러리 및 프레임워크
  ──────────────────────────────────────────────────────
  □ ARM64용 C++ MFC(MSVC 미리 보기)
  □ ARM64용 Spectre 완화 기능이 포함된 C++ MFC(MSVC 미리 보기)
  □ x64/x86용 C++ MFC(MSVC 미리 보기)
  □ x64/x86용 Spectre 완화 기능이 적용된 C++ MFC (MSVC 미리 보기)
```

**일반적인 Windows x64/x86 데스크톱 개발의 경우 권장 선택:**

| 항목 | 선택 이유 |
|------|-----------|
| `x64/x86용 C++ MFC(최신 MSVC)` | 최신 컴파일러 기반, 신규 프로젝트에 권장 |
| `x64/x86용 C++ MFC(MSVC v14.50)` | 특정 버전 고정이 필요할 때 (기업 환경 등) |

> **Spectre 완화 항목은 선택하지 않아도 됩니다.**  
> Spectre 완화(Spectre Mitigation)는 CPU 취약점 대응용으로,  
> 일반 학습/실습 환경에서는 불필요하며 빌드 속도가 느려질 수 있습니다.

> **ARM64 항목은 선택하지 않아도 됩니다.**  
> ARM64는 ARM 기반 Windows 장치(Surface Pro X 등) 대상 개발 시에만 필요합니다.

```
[수정] 클릭 → 설치 완료 대기 (수백 MB, 수 분 소요)
```

> 🔄 **VS 2022와의 차이**  
> VS 2026은 **빌드 도구(Build Tools)가 IDE와 분리**되어 있습니다.  
> IDE는 월별 자동 업데이트가 되고, 컴파일러/빌드 도구는 별도로 안정적으로 유지됩니다.  
> 따라서 IDE를 업데이트해도 기존 프로젝트의 빌드 환경이 깨지지 않습니다.  
> 또한 VS 2026은 `최신 MSVC`와 `v14.50` 두 가지 버전을 병행 제공하여  
> 프로젝트별로 컴파일러 버전을 선택할 수 있습니다.

---

### 2.3 프로젝트 생성 절차

**① VS 2026 시작 화면에서 새 프로젝트 만들기**

```
방법 A: 시작 화면(Start Window) → [새 프로젝트 만들기] 클릭
방법 B: 이미 VS가 열려 있다면 → 파일 → 새로 만들기 → 프로젝트  (Ctrl+Shift+N)
```

> 🔄 **VS 2022와의 차이**  
> VS 2026의 시작 화면은 Fluent UI로 재설계되어 더 깔끔하게 바뀌었지만  
> `파일 → 새로 만들기 → 프로젝트` 경로는 동일합니다.

**② 새 프로젝트 만들기 대화상자**

```
상단 필터 드롭다운:
  언어:    C++
  플랫폼:  Windows
  프로젝트 형식:  데스크톱

검색창에 "MFC 앱" 입력
→ 목록에서 [MFC 앱] 선택
→ [다음] 클릭
```

> 🔄 **VS 2022와의 차이**  
> 검색창 결과 카드 디자인이 Fluent 스타일로 바뀌었습니다.  
> 아이콘과 카드 레이아웃이 다를 수 있지만 "MFC 앱" 텍스트로 검색하면 동일하게 찾을 수 있습니다.

**③ 새 프로젝트 구성 화면**

```
프로젝트 이름:  CircleFromPoints
위치:          원하는 경로 선택
솔루션 이름:   CircleFromPoints  (자동 입력)
[만들기] 클릭
```

**④ MFC 앱 마법사 (애플리케이션 종류 설정)**

```
응용 프로그램 종류: ● 대화 상자 기반   ← 반드시 선택
대화 상자 제목:    세 점의 외접원
나머지 옵션:       기본값 유지
[마침] 클릭
```

> 🔄 **VS 2022와의 차이**  
> MFC 앱 마법사의 내부 UI 스타일이 Fluent 기반으로 업데이트되었습니다.  
> 하지만 설정 항목(응용 프로그램 종류, 대화상자 제목 등)의 구성은 동일합니다.

**⑤ 자동 생성 파일 확인**

```
CircleFromPoints.h / .cpp         ← 앱 클래스
CircleFromPointsDlg.h / .cpp      ← 대화상자 클래스  (주 작업 파일)
CircleFromPoints.rc               ← 리소스
pch.h                             ← 미리 컴파일된 헤더
```

---

### 2.4 VS 2026 솔루션 탐색기 사용법

> 🔄 **VS 2022와의 차이**  
> VS 2026은 솔루션 탐색기 항목 간격이 기본적으로 넓어졌습니다(접근성 향상).  
> 더 촘촘하게 보려면 아래 설정을 변경하세요.

```
도구 → 옵션 → 환경 → Visual Experience
  → "솔루션 탐색기에서 간격 줄이기(Use compact spacing)" 체크
```

---

### 2.5 VS 2026 설정 시스템 변경사항

> 🔄 **VS 2022와의 차이 (중요)**  
> VS 2026은 기존 `도구 → 옵션` 대화상자를 **새로운 모던 설정 UI**로 대체했습니다.

| 항목 | VS 2022 | VS 2026 |
|------|---------|---------|
| 설정 진입 | `도구 → 옵션` | `도구 → 옵션` (동일, but Fluent UI로 재설계) |
| 설정 검색 | 기본 검색 | AI 보조 검색 (더 정확한 결과) |
| 설정 저장 | 내부 바이너리 | JSON 파일로 변경 → 변경사항 실시간 추적 가능 |
| 미전환 설정 | — | **More Settings** 노드에서 레거시 설정 접근 가능 |
| 설정 동기화 | 이전 버전으로 동기화 가능 | **이전 버전으로 역동기화 불가** (신버전 → 구버전 방향 없음) |

**프로젝트 문자 집합 설정 경로 (VS 2026)**

```
방법 A (추천): 솔루션 탐색기에서 프로젝트 우클릭
  → 속성 → 구성 속성 → 고급 → 문자 집합
  → "유니코드 문자 집합 사용" 선택

방법 B: 상단 메뉴 → 프로젝트 → [프로젝트명] 속성
  → 구성 속성 → 고급 → 문자 집합 → 유니코드
```

---

### 2.6 리소스 편집기에서 기본 버튼 정리

```
솔루션 탐색기 → 리소스 파일 폴더 → CircleFromPoints.rc 더블클릭
  → 리소스 뷰(Resource View) 패널 열림
  → Dialog → IDD_CIRCLEFROMPOINTS_DIALOG 더블클릭
  → 기본 [확인], [취소] 버튼 선택 후 Delete
  → 대화상자 크기: 약 900×680 으로 조절 (테두리 드래그)
```

> 🔄 **VS 2022와의 차이**  
> 리소스 뷰(Resource View) 패널을 여는 방법:
> - VS 2022: `보기 → 다른 창 → 리소스 뷰` 또는 `Ctrl+Shift+E`
> - VS 2026: `보기 → 리소스 뷰` (메뉴 구조 단순화) 또는 동일하게 `Ctrl+Shift+E`

---

## 3. STEP 01 — 빈 대화상자에 마우스 클릭으로 점 찍기

### 3.1 목표

- 캔버스 영역에 좌클릭 시 빨간 점이 찍힘
- 점 찍을 때마다 화면 갱신

### 3.2 헤더 수정 (`CircleFromPointsDlg.h`)

```cpp
class CCircleFromPointsDlg : public CDialogEx
{
public:
    CCircleFromPointsDlg(CWnd* pParent = nullptr);
    virtual ~CCircleFromPointsDlg();

#ifdef AFX_DESIGN_TIME
    enum { IDD = IDD_CIRCLEFROMPOINTS_DIALOG };
#endif

protected:
    virtual void DoDataExchange(CDataExchange* pDX);
    virtual BOOL OnInitDialog();

    // ── [STEP01] 추가 ──────────────────────────
    CPoint  m_pts[3];       // 클릭 지점 저장
    int     m_nCount;       // 현재 점 개수

    void DrawScene(CDC* pDC);   // 그리기 전담 함수

    HICON   m_hIcon;

    afx_msg void OnPaint();
    afx_msg void OnSysCommand(UINT nID, LPARAM lParam);
    afx_msg HCURSOR OnQueryDragIcon();
    afx_msg void OnLButtonDown(UINT nFlags, CPoint point);  // ← 추가

    DECLARE_MESSAGE_MAP()
};
```

### 3.3 메시지 맵에 핸들러 등록 (`CircleFromPointsDlg.cpp`)

```cpp
BEGIN_MESSAGE_MAP(CCircleFromPointsDlg, CDialogEx)
    ON_WM_SYSCOMMAND()
    ON_WM_PAINT()
    ON_WM_QUERYDRAGICON()
    ON_WM_LBUTTONDOWN()     // ← 추가
END_MESSAGE_MAP()
```

### 3.4 생성자 초기화

```cpp
CCircleFromPointsDlg::CCircleFromPointsDlg(CWnd* pParent)
    : CDialogEx(IDD_CIRCLEFROMPOINTS_DIALOG, pParent)
    , m_nCount(0)           // ← 추가
{
    m_hIcon = AfxGetApp()->LoadIcon(IDR_MAINFRAME);
}
```

### 3.5 OnLButtonDown 구현

```cpp
void CCircleFromPointsDlg::OnLButtonDown(UINT nFlags, CPoint point)
{
    // 캔버스 영역(x < 700)만 처리
    if (point.x >= 700) {
        CDialogEx::OnLButtonDown(nFlags, point);
        return;
    }

    if (m_nCount < 3) {
        m_pts[m_nCount++] = point;
        Invalidate(FALSE);  // 화면 갱신 (FALSE: 배경 지우지 않음)
    }

    CDialogEx::OnLButtonDown(nFlags, point);
}
```

### 3.6 OnPaint — 더블 버퍼링 적용

> 더블 버퍼링: 메모리 DC에 먼저 그린 뒤 화면에 한 번에 복사 → 깜빡임 방지

```cpp
void CCircleFromPointsDlg::OnPaint()
{
    if (IsIconic()) {
        // 최소화 상태 처리 (기본 코드 유지)
        CPaintDC dc(this);
        // ... 생략
        return;
    }

    CPaintDC dc(this);
    CRect cr;
    GetClientRect(&cr);

    // ① 메모리 DC 생성
    CDC memDC;
    memDC.CreateCompatibleDC(&dc);

    CBitmap bmp;
    bmp.CreateCompatibleBitmap(&dc, cr.Width(), cr.Height());
    CBitmap* pOld = memDC.SelectObject(&bmp);

    // ② 배경 초기화
    memDC.FillSolidRect(cr, ::GetSysColor(COLOR_BTNFACE));

    // ③ 실제 그리기
    DrawScene(&memDC);

    // ④ 화면에 복사
    dc.BitBlt(0, 0, cr.Width(), cr.Height(), &memDC, 0, 0, SRCCOPY);
    memDC.SelectObject(pOld);
}
```

### 3.7 DrawScene — 점만 그리기 (STEP01 버전)

```cpp
void CCircleFromPointsDlg::DrawScene(CDC* pDC)
{
    pDC->SetBkMode(TRANSPARENT);

    // 캔버스 배경
    CRect canvas(0, 0, 700, 650);
    pDC->FillSolidRect(canvas, RGB(245, 247, 252));

    // 안내 메시지
    CString msg;
    msg.Format(_T("좌클릭으로 점을 찍으세요 (%d/3)"), m_nCount);
    pDC->SetTextColor(RGB(80, 80, 80));
    pDC->TextOut(10, 10, msg);

    // 점 그리기 (임시: Rectangle로 표시)
    for (int i = 0; i < m_nCount; i++) {
        int x = m_pts[i].x, y = m_pts[i].y;
        CBrush br(RGB(220, 50, 50));
        pDC->SelectObject(&br);
        pDC->Rectangle(x-5, y-5, x+5, y+5);   // 나중에 원으로 교체
    }
}
```

> ⚠ 이 단계에서는 `Rectangle`로 임시 표시합니다. STEP 05에서 `Polyline` 원으로 교체합니다.

---

## 4. STEP 02 — 3점 완성 시 외접원 계산 및 표시

### 4.1 목표

- 3번째 점이 찍히면 자동으로 외접원 계산
- 일직선이면 오류 메시지 표시
- 외접원을 화면에 그리기

### 4.2 헤더에 변수 추가

```cpp
// ── [STEP02] 추가 ──────────────────────────────────
bool    m_bCircleReady;     // 외접원 계산 완료 여부
bool    m_bCannotDraw;      // 일직선 여부 (원 불가)
double  m_cx, m_cy;         // 외접원 중심
double  m_radius;           // 외접원 반지름

void CalcCircumCircle();    // 외접원 계산 함수
```

### 4.3 생성자 초기화 추가

```cpp
, m_bCircleReady(false)
, m_bCannotDraw(false)
, m_cx(0), m_cy(0), m_radius(0)
```

### 4.4 CalcCircumCircle 구현

```cpp
void CCircleFromPointsDlg::CalcCircumCircle()
{
    double x1=m_pts[0].x, y1=m_pts[0].y;
    double x2=m_pts[1].x, y2=m_pts[1].y;
    double x3=m_pts[2].x, y3=m_pts[2].y;

    // 분모 D (= 2 × 삼각형 넓이)
    double D = 2.0*(x1*(y2-y3) + x2*(y3-y1) + x3*(y1-y2));

    if (fabs(D) < 1e-6) {       // 일직선 판별
        m_bCannotDraw  = true;
        m_bCircleReady = false;
        return;
    }

    double s1 = x1*x1+y1*y1;
    double s2 = x2*x2+y2*y2;
    double s3 = x3*x3+y3*y3;

    m_cx = (s1*(y2-y3) + s2*(y3-y1) + s3*(y1-y2)) / D;
    m_cy = (s1*(x3-x2) + s2*(x1-x3) + s3*(x2-x1)) / D;
    m_radius = sqrt((x1-m_cx)*(x1-m_cx) + (y1-m_cy)*(y1-m_cy));

    m_bCircleReady = true;
    m_bCannotDraw  = false;
}
```

### 4.5 OnLButtonDown 수정 — 3번째 점에서 계산 호출

```cpp
void CCircleFromPointsDlg::OnLButtonDown(UINT nFlags, CPoint point)
{
    if (point.x >= 700) { CDialogEx::OnLButtonDown(nFlags,point); return; }

    if (m_nCount < 3) {
        m_pts[m_nCount++] = point;

        if (m_nCount == 3)          // ← 추가: 3번째 점에서 계산
            CalcCircumCircle();

        Invalidate(FALSE);
    }
    CDialogEx::OnLButtonDown(nFlags, point);
}
```

### 4.6 DrawScene 수정 — 외접원 및 오류 메시지 추가

```cpp
// 외접원 (임시: Ellipse 사용 — STEP05에서 Polyline으로 교체 예정)
if (m_bCircleReady) {
    CPen pen(PS_SOLID, 2, RGB(30, 100, 220));
    pDC->SelectObject(&pen);
    pDC->SelectStockObject(NULL_BRUSH);
    int L=(int)(m_cx-m_radius), T=(int)(m_cy-m_radius);
    int R=(int)(m_cx+m_radius), B=(int)(m_cy+m_radius);
    pDC->Ellipse(L, T, R, B);  // ← STEP05에서 교체
}

// 오류 메시지
if (m_bCannotDraw) {
    pDC->SetTextColor(RGB(200, 30, 30));
    pDC->TextOut(10, 10, _T("⚠ 세 점이 일직선 — 원을 그릴 수 없습니다."));
}
```

---

## 5. STEP 03 — 계산 과정 시각화 (수직이등분선 등)

### 5.1 목표

- 선분 P₁P₂, P₂P₃ 표시
- 각 선분의 중점(◆) 표시
- 중점에서 수직이등분선(점선) 화면 끝까지 연장
- 수직 기호(⊾) 표시
- 중심→각 점 반지름선(점선) 표시

### 5.2 DrawScene에 추가할 코드

#### 선분 그리기

```cpp
if (m_nCount >= 2) {
    CPen segPen(PS_SOLID, 1, RGB(190, 190, 190));
    pDC->SelectObject(&segPen);
    pDC->MoveTo(m_pts[0]); pDC->LineTo(m_pts[1]);
    if (m_nCount == 3) {
        pDC->MoveTo(m_pts[1]); pDC->LineTo(m_pts[2]);
    }
}
```

#### 수직이등분선 그리기 (람다 함수로 재사용)

```cpp
// 수직이등분선을 그리는 람다
auto drawPerp = [&](CPoint A, CPoint B, COLORREF col, LPCTSTR lbl)
{
    // 중점 계산
    double mx = (A.x+B.x)/2.0, my = (A.y+B.y)/2.0;

    // 선분 방향벡터 → 수직벡터
    double dx = B.x-A.x, dy = B.y-A.y;
    double len = sqrt(dx*dx + dy*dy);
    if (len < 1e-6) return;

    double px = -dy/len, py = dx/len;  // 수직이등분선 방향 (정규화)

    // 화면 대각선 길이만큼 연장
    double ext = sqrt(700.0*700.0 + 650.0*650.0);

    // 점선으로 그리기
    CPen dPen(PS_DASH, 1, col);
    pDC->SelectObject(&dPen);
    pDC->SelectStockObject(NULL_BRUSH);
    pDC->MoveTo((int)(mx - px*ext), (int)(my - py*ext));
    pDC->LineTo((int)(mx + px*ext), (int)(my + py*ext));

    // 중점 마름모(◆) 표시
    int qs = 6;
    POINT dm[5] = {
        {(int)mx,    (int)my-qs},
        {(int)mx+qs, (int)my   },
        {(int)mx,    (int)my+qs},
        {(int)mx-qs, (int)my   },
        {(int)mx,    (int)my-qs}
    };
    CPen sPen(PS_SOLID, 1, col);
    pDC->SelectObject(&sPen);
    pDC->Polyline(dm, 5);

    // 수직 기호(⊾) — 작은 사각형으로 표현
    double tx = dx/len, ty = dy/len;    // 선분 방향 (정규화)
    int q = 7;
    POINT sq[5] = {
        {(int)(mx+px*q),        (int)(my+py*q)       },
        {(int)(mx+px*q+tx*q),   (int)(my+py*q+ty*q)  },
        {(int)(mx+tx*q),        (int)(my+ty*q)        },
        {(int)mx,               (int)my               },
        {(int)(mx+px*q),        (int)(my+py*q)        }
    };
    pDC->Polyline(sq, 5);

    // 레이블
    pDC->SetTextColor(col);
    pDC->TextOut((int)(mx+px*40)+4, (int)(my+py*40)-14, lbl);
};

if (m_nCount >= 2) {
    drawPerp(m_pts[0], m_pts[1], RGB(220,120,0), _T("L₁"));
    if (m_nCount == 3)
        drawPerp(m_pts[1], m_pts[2], RGB(140,0,200), _T("L₂"));
}
```

#### 반지름선 그리기

```cpp
if (m_bCircleReady) {
    CPen rPen(PS_DOT, 1, RGB(120, 160, 255));
    pDC->SelectObject(&rPen);
    for (int i = 0; i < 3; i++) {
        pDC->MoveTo((int)m_cx, (int)m_cy);
        pDC->LineTo(m_pts[i]);
    }
    // 반지름 레이블
    double mx = (m_cx+m_pts[0].x)/2.0;
    double my = (m_cy+m_pts[0].y)/2.0;
    CString rl;
    rl.Format(_T("r=%.1f"), m_radius);
    pDC->SetTextColor(RGB(30,100,220));
    pDC->TextOut((int)mx+3, (int)my-13, rl);
}
```

---

## 6. STEP 04 — 사용자 입력: 점 반지름 / 외접원 두께

### 6.1 목표

- 우측 패널에 `CEdit` 컨트롤 동적 생성
- 클릭 지점 원 반지름, 외접원 두께를 입력받아 그리기에 반영

### 6.2 헤더에 추가

```cpp
// ── [STEP04] 추가 ──────────────────────────────────
int     m_nDotRadius;       // 클릭 지점 원 반지름
int     m_nThickness;       // 외접원 테두리 두께

CEdit   m_editRadius;       // 입력 컨트롤
CEdit   m_editThickness;
CStatic m_staticInfo;       // 정보 표시

// 리소스 ID 정의 (헤더 상단 또는 별도 헤더)
#define IDC_EDIT_RADIUS    1003
#define IDC_EDIT_THICKNESS 1004
#define IDC_STATIC_INFO    1005
```

### 6.3 OnInitDialog에서 컨트롤 동적 생성

```cpp
BOOL CCircleFromPointsDlg::OnInitDialog()
{
    CDialogEx::OnInitDialog();
    // ... 기존 코드 ...

    // 창 크기 설정
    MoveWindow(100, 50, 900, 680, TRUE);

    // 그룹박스
    CButton* pGrp = new CButton();
    pGrp->Create(_T("설정"), WS_CHILD|WS_VISIBLE|BS_GROUPBOX,
                 CRect(700, 5, 895, 245), this, 9999);

    // 점 원 반지름 입력
    CStatic* pL1 = new CStatic();
    pL1->Create(_T("클릭점 원 반지름"), WS_CHILD|WS_VISIBLE|SS_LEFT,
                CRect(710, 20, 885, 38), this);
    m_editRadius.Create(WS_CHILD|WS_VISIBLE|WS_BORDER|ES_NUMBER,
                        CRect(710, 42, 800, 62), this, IDC_EDIT_RADIUS);
    m_editRadius.SetWindowText(_T("10"));

    // 외접원 두께 입력
    CStatic* pL2 = new CStatic();
    pL2->Create(_T("외접원 테두리 두께"), WS_CHILD|WS_VISIBLE|SS_LEFT,
                CRect(710, 80, 885, 98), this);
    m_editThickness.Create(WS_CHILD|WS_VISIBLE|WS_BORDER|ES_NUMBER,
                           CRect(710, 100, 800, 120), this, IDC_EDIT_THICKNESS);
    m_editThickness.SetWindowText(_T("2"));

    // 정보 표시 영역
    m_staticInfo.Create(_T(""), WS_CHILD|WS_VISIBLE|SS_LEFT,
                        CRect(710, 260, 890, 530), this, IDC_STATIC_INFO);

    return TRUE;
}
```

### 6.4 DrawScene에서 입력값 읽기

```cpp
// DrawScene() 시작 부분에서 항상 입력값 읽기
CString sR, sT;
m_editRadius.GetWindowText(sR);
m_editThickness.GetWindowText(sT);
m_nDotRadius = max(3,  _ttoi(sR));
m_nThickness = max(1,  _ttoi(sT));
```

### 6.5 정보 패널 업데이트 함수

```cpp
void UpdateInfoPanel(CStatic& st, CPoint pts[], int n,
                     bool ready, bool cannot,
                     double cx, double cy, double r)
{
    CString s;
    for (int i = 0; i < n; i++)
        s.AppendFormat(_T("P%d : (%d, %d)\r\n"), i+1, pts[i].x, pts[i].y);

    if (ready)
        s.AppendFormat(_T("\r\n외접원 중심\r\nO(%.1f, %.1f)\r\n반지름 r=%.1f"),
                       cx, cy, r);
    else if (cannot)
        s += _T("\r\n⚠ 일직선: 원 불가");

    st.SetWindowText(s);
}
```

---

## 7. STEP 05 — Ellipse API 금지: Polyline으로 원 그리기

### 7.1 목표

> `Ellipse`, `GDI+`, `wingdi`, `FillPolygon` 등 사용 금지  
> 삼각함수로 원 위의 점들을 계산하고 `Polyline`으로 연결

### 7.2 원리

```
원 위의 점: (cx + r·cos(θ), cy + r·sin(θ))
θ = 0, 2π/N, 4π/N, ..., 2π   (N개의 분할)
이 점들을 순서대로 Polyline으로 연결 → 원 근사
```

분할 수 N이 클수록 더 매끄러운 원이 됩니다.

### 7.3 DrawCircleByLine 구현

```cpp
#include <vector>
#ifndef M_PI
#define M_PI 3.14159265358979323846
#endif

// 외접선으로만 이루어진 원 (내부 투명)
void CCircleFromPointsDlg::DrawCircleByLine(
    CDC* pDC, double cx, double cy, double r, int segments)
{
    if (r < 1.0) return;

    std::vector<POINT> pts;
    pts.reserve(segments + 1);

    for (int i = 0; i <= segments; i++) {
        double angle = 2.0 * M_PI * i / segments;
        POINT p;
        p.x = (int)(cx + r * cos(angle));
        p.y = (int)(cy + r * sin(angle));
        pts.push_back(p);
    }
    // 마지막 점 = 첫 점 → 닫힌 원
    pDC->Polyline(pts.data(), (int)pts.size());
}
```

### 7.4 속 채운 원: DrawFilledCircleByLine

```cpp
// 속이 채워진 원 (클릭 지점 표시용)
// 반지름을 0.8씩 증가시키며 여러 겹의 원을 그려 채움
void CCircleFromPointsDlg::DrawFilledCircleByLine(
    CDC* pDC, double cx, double cy, double r)
{
    for (double ri = 1.0; ri <= r; ri += 0.8)
        DrawCircleByLine(pDC, cx, cy, ri, 180);
}
```

### 7.5 두께 있는 외접원: DrawThickCircle

```cpp
// 외접원 두께 구현: 반지름을 ±offset 조정하며 여러 겹 그리기
// 내부는 투명 유지 (속 채우지 않음)
for (int t = 0; t < m_nThickness; t++) {
    double offset = t - m_nThickness / 2.0;
    CPen cPen(PS_SOLID, 1, RGB(30, 100, 220));
    pDC->SelectObject(&cPen);
    pDC->SelectStockObject(NULL_BRUSH);
    DrawCircleByLine(pDC, m_cx, m_cy, m_radius + offset);
}
```

### 7.6 DrawScene에서 기존 Ellipse 호출 → DrawCircleByLine으로 교체

```cpp
// 이전 (STEP02):
// pDC->Ellipse(L, T, R, B);   ← 삭제

// 변경 후 (STEP05):
for (int t = 0; t < m_nThickness; t++) {
    double offset = t - m_nThickness / 2.0;
    CPen cPen(PS_SOLID, 1, RGB(30, 100, 220));
    pDC->SelectObject(&cPen);
    pDC->SelectStockObject(NULL_BRUSH);
    DrawCircleByLine(pDC, m_cx, m_cy, m_radius + offset);
}
```

---

## 8. STEP 06 — 점 드래그로 외접원 실시간 재계산

### 8.1 목표

- 3개의 점 중 하나를 클릭하고 드래그하면 점이 이동
- 드래그 중 매 마우스 이동 이벤트마다 외접원 재계산 및 화면 갱신
- 4번째 이후 클릭은 점 원을 그리지 않음

### 8.2 헤더에 추가

```cpp
// ── [STEP06] 추가 ──────────────────────────────────
int     m_nDragIdx;     // 드래그 중인 점 인덱스 (-1: 없음)
bool    m_bDragging;    // 드래그 상태 플래그

bool HitTestPoint(CPoint pt, int& outIdx);  // 히트 테스트

// 메시지 핸들러 선언
afx_msg void OnLButtonUp(UINT nFlags, CPoint point);
afx_msg void OnMouseMove(UINT nFlags, CPoint point);
```

### 8.3 메시지 맵에 추가

```cpp
BEGIN_MESSAGE_MAP(CCircleFromPointsDlg, CDialogEx)
    // ... 기존 ...
    ON_WM_LBUTTONUP()       // ← 추가
    ON_WM_MOUSEMOVE()       // ← 추가
END_MESSAGE_MAP()
```

### 8.4 히트 테스트 구현

```cpp
// 클릭한 위치가 기존 점 위인지 확인
bool CCircleFromPointsDlg::HitTestPoint(CPoint pt, int& outIdx)
{
    int hitR = max(m_nDotRadius + 4, 12);   // 히트 영역 = 원 반지름 + 여유
    for (int i = 0; i < 3 && i < m_nCount; i++) {
        int dx = pt.x - m_pts[i].x;
        int dy = pt.y - m_pts[i].y;
        if (dx*dx + dy*dy <= hitR*hitR) {
            outIdx = i;
            return true;
        }
    }
    return false;
}
```

### 8.5 OnLButtonDown 수정 — 드래그 감지 추가

```cpp
void CCircleFromPointsDlg::OnLButtonDown(UINT nFlags, CPoint point)
{
    if (point.x >= 700) { CDialogEx::OnLButtonDown(nFlags,point); return; }

    // 3점 완성 후 드래그 히트 테스트
    if (m_nCount == 3 && m_bCircleReady) {
        int idx = -1;
        if (HitTestPoint(point, idx)) {
            m_bDragging = true;
            m_nDragIdx  = idx;
            SetCapture();       // 마우스 캡처: 창 밖으로 나가도 이벤트 수신
            CDialogEx::OnLButtonDown(nFlags, point);
            return;
        }
        // 히트 없으면 새 점 입력 시작
        m_nCount = 0;
        m_bCircleReady = false;
        m_bCannotDraw  = false;
    }

    if (m_nCount < 3) {
        m_pts[m_nCount++] = point;
        if (m_nCount == 3) CalcCircumCircle();
        Invalidate(FALSE);
    }
    CDialogEx::OnLButtonDown(nFlags, point);
}
```

### 8.6 OnMouseMove — 드래그 중 실시간 갱신

```cpp
void CCircleFromPointsDlg::OnMouseMove(UINT nFlags, CPoint point)
{
    if (m_bDragging && m_nDragIdx >= 0) {
        // 캔버스 경계 클램프
        point.x = max(1, min(point.x, 699));
        point.y = max(1, min(point.y, 639));

        m_pts[m_nDragIdx] = point;  // 점 이동
        CalcCircumCircle();          // 외접원 재계산
        Invalidate(FALSE);           // 화면 즉시 갱신
    }
    CDialogEx::OnMouseMove(nFlags, point);
}
```

### 8.7 OnLButtonUp — 드래그 종료

```cpp
void CCircleFromPointsDlg::OnLButtonUp(UINT nFlags, CPoint point)
{
    if (m_bDragging) {
        m_bDragging = false;
        m_nDragIdx  = -1;
        ReleaseCapture();   // 마우스 캡처 해제
    }
    CDialogEx::OnLButtonUp(nFlags, point);
}
```

### 8.8 DrawScene — 4번째 클릭 이후 점 원 미표시

```cpp
// min(m_nCount, 3) 범위만 루프
// → 3점 완성 후 새 클릭이 와도 점 원은 3개까지만 표시
for (int i = 0; i < min(m_nCount, 3); i++) {
    int x = m_pts[i].x, y = m_pts[i].y;

    CPen dotPen(PS_SOLID, 1, RGB(200, 40, 40));
    pDC->SelectObject(&dotPen);
    DrawFilledCircleByLine(pDC, x, y, m_nDotRadius);   // Polyline 원

    // 좌표 레이블
    CString lb;
    lb.Format(_T("P%d(%d,%d)"), i+1, x, y);
    pDC->SetTextColor(RGB(160, 20, 20));
    pDC->TextOut(x + m_nDotRadius + 3, y - 14, lb);
}
```

---

## 9. STEP 07 — 초기화 버튼

### 9.1 목표

- [초기화] 버튼 클릭 시 모든 상태 초기화
- 캔버스 완전히 클리어

### 9.2 헤더에 추가

```cpp
CButton m_btnReset;

#define IDC_BTN_RESET  1001

afx_msg void OnBtnReset();
```

### 9.3 메시지 맵에 추가

```cpp
ON_BN_CLICKED(IDC_BTN_RESET, &CCircleFromPointsDlg::OnBtnReset)
```

### 9.4 OnInitDialog에서 버튼 생성

```cpp
m_btnReset.Create(_T("초기화"),
                  WS_CHILD|WS_VISIBLE|BS_PUSHBUTTON,
                  CRect(710, 150, 870, 180), this, IDC_BTN_RESET);
```

### 9.5 OnBtnReset 구현

```cpp
void CCircleFromPointsDlg::OnBtnReset()
{
    // 랜덤 쓰레드 중단 (STEP08 추가 후 유효)
    m_bThreadRun = false;
    if (m_pThread) {
        WaitForSingleObject(m_pThread->m_hThread, 3000);
        m_pThread = nullptr;
    }

    // 모든 상태 초기화
    m_nCount       = 0;
    m_bCircleReady = false;
    m_bCannotDraw  = false;
    m_bDragging    = false;
    m_nDragIdx     = -1;

    Invalidate(FALSE);
}
```

---

## 10. STEP 08 — 랜덤 이동 (별도 쓰레드, 프리징 없음)

### 10.1 목표

- [랜덤 이동] 버튼: 3개 점을 랜덤 위치로 이동, 외접원 재계산
- 초당 2회(500ms 간격), 총 10회 반복
- 메인 UI 프리징 없음 → **별도 쓰레드 + `PostMessage`** 패턴

### 10.2 왜 PostMessage인가?

```
[작업 쓰레드]                    [메인 UI 쓰레드]
    │                                  │
    │  점 좌표 변경                    │
    │  CalcCircumCircle()              │
    │                                  │
    │──PostMessage(WM_REDRAW)─────────▶│
    │                                  │  Invalidate() 호출
    │  Sleep(500ms)                    │  UpdateWindow()
    │                                  │  → OnPaint() 실행
```

> MFC에서 UI 컨트롤(`Invalidate`, `SetWindowText` 등)은 반드시 메인 쓰레드에서만 호출해야 합니다.  
> 작업 쓰레드에서 직접 UI를 조작하면 충돌(Crash)이 발생할 수 있습니다.

### 10.3 헤더에 추가

```cpp
// ── [STEP08] 추가 ──────────────────────────────────
CButton         m_btnRandom;
CWinThread*     m_pThread;
volatile bool   m_bThreadRun;

static UINT RandomThreadProc(LPVOID pParam);

#define IDC_BTN_RANDOM     1002
#define WM_REDRAW_REQUEST  (WM_USER + 1)

afx_msg void OnBtnRandom();
afx_msg LRESULT OnRedrawRequest(WPARAM wParam, LPARAM lParam);
```

### 10.4 메시지 맵에 추가

```cpp
ON_BN_CLICKED(IDC_BTN_RANDOM, &CCircleFromPointsDlg::OnBtnRandom)
ON_MESSAGE(WM_REDRAW_REQUEST,  &CCircleFromPointsDlg::OnRedrawRequest)
```

### 10.5 OnInitDialog에서 버튼 생성

```cpp
m_btnRandom.Create(_T("랜덤 이동"),
                   WS_CHILD|WS_VISIBLE|BS_PUSHBUTTON,
                   CRect(710, 195, 870, 225), this, IDC_BTN_RANDOM);
```

### 10.6 OnBtnRandom 구현

```cpp
void CCircleFromPointsDlg::OnBtnRandom()
{
    if (!m_bCircleReady) {
        AfxMessageBox(_T("먼저 세 점을 찍어 원을 완성하세요."));
        return;
    }
    if (m_bThreadRun) return;   // 이미 실행 중이면 무시

    m_bThreadRun = true;
    m_pThread = AfxBeginThread(
        RandomThreadProc,           // 쓰레드 함수
        this,                       // 파라미터 (this 포인터)
        THREAD_PRIORITY_NORMAL,
        0, 0, nullptr
    );
}
```

### 10.7 랜덤 쓰레드 함수 구현

```cpp
UINT CCircleFromPointsDlg::RandomThreadProc(LPVOID pParam)
{
    CCircleFromPointsDlg* pDlg =
        reinterpret_cast<CCircleFromPointsDlg*>(pParam);

    const int CANVAS_W = 698;
    const int CANVAS_H = 638;
    const int MARGIN   = 30;

    for (int i = 0; i < 10 && pDlg->m_bThreadRun; i++)
    {
        // 랜덤 좌표로 3점 이동
        for (int j = 0; j < 3; j++) {
            pDlg->m_pts[j].x = MARGIN + rand() % (CANVAS_W - MARGIN*2);
            pDlg->m_pts[j].y = MARGIN + rand() % (CANVAS_H - MARGIN*2);
        }

        // 외접원 재계산
        pDlg->CalcCircumCircle();

        // ★ PostMessage: 메인 쓰레드에 화면 갱신 요청
        //   (작업 쓰레드에서 직접 Invalidate() 호출 금지)
        pDlg->PostMessage(WM_REDRAW_REQUEST, 0, 0);

        Sleep(500);     // 0.5초 대기 = 초당 2회
    }

    pDlg->m_bThreadRun = false;
    return 0;
}
```

### 10.8 OnRedrawRequest 구현 (메인 쓰레드에서 실행됨)

```cpp
LRESULT CCircleFromPointsDlg::OnRedrawRequest(WPARAM, LPARAM)
{
    Invalidate(FALSE);      // 화면 갱신 요청
    UpdateWindow();         // 즉시 WM_PAINT 처리
    return 0;
}
```

### 10.9 생성자 초기화 추가

```cpp
, m_pThread(nullptr)
, m_bThreadRun(false)
```

---

## 11. 전체 소스 코드

### CircleFromPointsDlg.h (최종)

```cpp
#pragma once
#include <vector>
#include <cmath>

#define IDC_BTN_RESET      1001
#define IDC_BTN_RANDOM     1002
#define IDC_EDIT_RADIUS    1003
#define IDC_EDIT_THICKNESS 1004
#define IDC_STATIC_INFO    1005
#define WM_REDRAW_REQUEST  (WM_USER + 1)

class CCircleFromPointsDlg : public CDialogEx
{
public:
    CCircleFromPointsDlg(CWnd* pParent = nullptr);
    virtual ~CCircleFromPointsDlg();

#ifdef AFX_DESIGN_TIME
    enum { IDD = IDD_CIRCLEFROMPOINTS_DIALOG };
#endif

protected:
    virtual void DoDataExchange(CDataExchange* pDX);
    virtual BOOL OnInitDialog();

    // ── 그리기 ───────────────────────────────────
    void DrawScene(CDC* pDC);
    void DrawCircleByLine(CDC* pDC, double cx, double cy,
                          double r, int segments = 360);
    void DrawFilledCircleByLine(CDC* pDC, double cx, double cy, double r);

    // ── 계산 ─────────────────────────────────────
    void CalcCircumCircle();
    bool HitTestPoint(CPoint pt, int& outIdx);

    // ── 상태 ─────────────────────────────────────
    CPoint  m_pts[3];
    int     m_nCount;
    bool    m_bCircleReady;
    bool    m_bCannotDraw;
    double  m_cx, m_cy, m_radius;
    int     m_nDotRadius;
    int     m_nThickness;

    // ── 드래그 ───────────────────────────────────
    int     m_nDragIdx;
    bool    m_bDragging;

    // ── 쓰레드 ───────────────────────────────────
    static UINT RandomThreadProc(LPVOID pParam);
    CWinThread*   m_pThread;
    volatile bool m_bThreadRun;

    // ── 컨트롤 ───────────────────────────────────
    HICON   m_hIcon;
    CButton m_btnReset;
    CButton m_btnRandom;
    CEdit   m_editRadius;
    CEdit   m_editThickness;
    CStatic m_staticInfo;

    // ── 메시지 핸들러 ────────────────────────────
    afx_msg void OnPaint();
    afx_msg void OnSysCommand(UINT nID, LPARAM lParam);
    afx_msg HCURSOR OnQueryDragIcon();
    afx_msg void OnLButtonDown(UINT nFlags, CPoint point);
    afx_msg void OnLButtonUp(UINT nFlags, CPoint point);
    afx_msg void OnMouseMove(UINT nFlags, CPoint point);
    afx_msg void OnBtnReset();
    afx_msg void OnBtnRandom();
    afx_msg LRESULT OnRedrawRequest(WPARAM wParam, LPARAM lParam);

    DECLARE_MESSAGE_MAP()
};
```

### CircleFromPointsDlg.cpp (최종)

```cpp
#include "pch.h"
#include "framework.h"
#include "CircleFromPoints.h"
#include "CircleFromPointsDlg.h"
#include "afxdialogex.h"
#include <cmath>
#include <vector>
#include <cstdlib>
#include <ctime>

#ifdef _DEBUG
#define new DEBUG_NEW
#endif

#ifndef M_PI
#define M_PI 3.14159265358979323846
#endif

// ── About 다이얼로그 ──────────────────────────────────
class CAboutDlg : public CDialogEx {
public:
    CAboutDlg() : CDialogEx(IDD_ABOUTBOX) {}
protected:
    void DoDataExchange(CDataExchange* pDX)
    { CDialogEx::DoDataExchange(pDX); }
    DECLARE_MESSAGE_MAP()
};
BEGIN_MESSAGE_MAP(CAboutDlg, CDialogEx)
END_MESSAGE_MAP()

// ── 메시지 맵 ─────────────────────────────────────────
BEGIN_MESSAGE_MAP(CCircleFromPointsDlg, CDialogEx)
    ON_WM_SYSCOMMAND()
    ON_WM_PAINT()
    ON_WM_QUERYDRAGICON()
    ON_WM_LBUTTONDOWN()
    ON_WM_LBUTTONUP()
    ON_WM_MOUSEMOVE()
    ON_BN_CLICKED(IDC_BTN_RESET,  &CCircleFromPointsDlg::OnBtnReset)
    ON_BN_CLICKED(IDC_BTN_RANDOM, &CCircleFromPointsDlg::OnBtnRandom)
    ON_MESSAGE(WM_REDRAW_REQUEST,  &CCircleFromPointsDlg::OnRedrawRequest)
END_MESSAGE_MAP()

// ── 생성자 / 소멸자 ───────────────────────────────────
CCircleFromPointsDlg::CCircleFromPointsDlg(CWnd* pParent)
    : CDialogEx(IDD_CIRCLEFROMPOINTS_DIALOG, pParent)
    , m_nCount(0), m_bCircleReady(false), m_bCannotDraw(false)
    , m_cx(0), m_cy(0), m_radius(0)
    , m_nDotRadius(10), m_nThickness(2)
    , m_nDragIdx(-1), m_bDragging(false)
    , m_pThread(nullptr), m_bThreadRun(false)
{
    m_hIcon = AfxGetApp()->LoadIcon(IDR_MAINFRAME);
    srand((unsigned)time(nullptr));
}

CCircleFromPointsDlg::~CCircleFromPointsDlg()
{
    m_bThreadRun = false;
}

void CCircleFromPointsDlg::DoDataExchange(CDataExchange* pDX)
{
    CDialogEx::DoDataExchange(pDX);
}

// ── OnInitDialog ──────────────────────────────────────
BOOL CCircleFromPointsDlg::OnInitDialog()
{
    CDialogEx::OnInitDialog();

    ASSERT((IDM_ABOUTBOX & 0xFFF0) == IDM_ABOUTBOX);
    ASSERT(IDM_ABOUTBOX < 0xF000);
    CMenu* pSysMenu = GetSystemMenu(FALSE);
    if (pSysMenu) {
        CString str; str.LoadString(IDS_ABOUTBOX);
        if (!str.IsEmpty()) {
            pSysMenu->AppendMenu(MF_SEPARATOR);
            pSysMenu->AppendMenu(MF_STRING, IDM_ABOUTBOX, str);
        }
    }
    SetIcon(m_hIcon, TRUE);
    SetIcon(m_hIcon, FALSE);
    SetWindowText(_T("세 점의 외접원"));
    MoveWindow(100, 50, 900, 680, TRUE);

    // 설정 그룹박스
    CButton* pGrp = new CButton();
    pGrp->Create(_T("설정"), WS_CHILD|WS_VISIBLE|BS_GROUPBOX,
                 CRect(700, 5, 895, 245), this, 9999);

    // 점 원 반지름
    (new CStatic())->Create(_T("클릭점 원 반지름"),
        WS_CHILD|WS_VISIBLE|SS_LEFT, CRect(710,20,885,38), this);
    m_editRadius.Create(WS_CHILD|WS_VISIBLE|WS_BORDER|ES_NUMBER,
        CRect(710,42,800,62), this, IDC_EDIT_RADIUS);
    m_editRadius.SetWindowText(_T("10"));

    // 외접원 두께
    (new CStatic())->Create(_T("외접원 테두리 두께"),
        WS_CHILD|WS_VISIBLE|SS_LEFT, CRect(710,80,885,98), this);
    m_editThickness.Create(WS_CHILD|WS_VISIBLE|WS_BORDER|ES_NUMBER,
        CRect(710,100,800,120), this, IDC_EDIT_THICKNESS);
    m_editThickness.SetWindowText(_T("2"));

    // 버튼
    m_btnReset.Create(_T("초기화"),
        WS_CHILD|WS_VISIBLE|BS_PUSHBUTTON,
        CRect(710,150,870,180), this, IDC_BTN_RESET);
    m_btnRandom.Create(_T("랜덤 이동"),
        WS_CHILD|WS_VISIBLE|BS_PUSHBUTTON,
        CRect(710,195,870,225), this, IDC_BTN_RANDOM);

    // 정보 패널
    m_staticInfo.Create(_T(""), WS_CHILD|WS_VISIBLE|SS_LEFT,
        CRect(710,260,890,530), this, IDC_STATIC_INFO);

    return TRUE;
}

void CCircleFromPointsDlg::OnSysCommand(UINT nID, LPARAM lParam)
{
    if ((nID & 0xFFF0) == IDM_ABOUTBOX)
        CAboutDlg().DoModal();
    else
        CDialogEx::OnSysCommand(nID, lParam);
}

HCURSOR CCircleFromPointsDlg::OnQueryDragIcon()
{
    return static_cast<HCURSOR>(m_hIcon);
}

// ── Polyline 원 그리기 ────────────────────────────────
void CCircleFromPointsDlg::DrawCircleByLine(
    CDC* pDC, double cx, double cy, double r, int segments)
{
    if (r < 1.0) return;
    std::vector<POINT> pts;
    pts.reserve(segments + 1);
    for (int i = 0; i <= segments; i++) {
        double a = 2.0 * M_PI * i / segments;
        POINT p = { (int)(cx + r*cos(a)), (int)(cy + r*sin(a)) };
        pts.push_back(p);
    }
    pDC->Polyline(pts.data(), (int)pts.size());
}

void CCircleFromPointsDlg::DrawFilledCircleByLine(
    CDC* pDC, double cx, double cy, double r)
{
    for (double ri = 1.0; ri <= r; ri += 0.8)
        DrawCircleByLine(pDC, cx, cy, ri, 180);
}

// ── 외접원 계산 ───────────────────────────────────────
void CCircleFromPointsDlg::CalcCircumCircle()
{
    double x1=m_pts[0].x, y1=m_pts[0].y;
    double x2=m_pts[1].x, y2=m_pts[1].y;
    double x3=m_pts[2].x, y3=m_pts[2].y;
    double D = 2.0*(x1*(y2-y3)+x2*(y3-y1)+x3*(y1-y2));
    if (fabs(D) < 1e-6) { m_bCannotDraw=true; m_bCircleReady=false; return; }
    double s1=x1*x1+y1*y1, s2=x2*x2+y2*y2, s3=x3*x3+y3*y3;
    m_cx = (s1*(y2-y3)+s2*(y3-y1)+s3*(y1-y2))/D;
    m_cy = (s1*(x3-x2)+s2*(x1-x3)+s3*(x2-x1))/D;
    m_radius = sqrt((x1-m_cx)*(x1-m_cx)+(y1-m_cy)*(y1-m_cy));
    m_bCircleReady=true; m_bCannotDraw=false;
}

// ── 히트 테스트 ───────────────────────────────────────
bool CCircleFromPointsDlg::HitTestPoint(CPoint pt, int& outIdx)
{
    int hitR = max(m_nDotRadius+4, 12);
    for (int i=0; i<3&&i<m_nCount; i++) {
        int dx=pt.x-m_pts[i].x, dy=pt.y-m_pts[i].y;
        if (dx*dx+dy*dy <= hitR*hitR) { outIdx=i; return true; }
    }
    return false;
}

// ── DrawScene ─────────────────────────────────────────
void CCircleFromPointsDlg::DrawScene(CDC* pDC)
{
    CString sR, sT;
    m_editRadius.GetWindowText(sR);
    m_editThickness.GetWindowText(sT);
    m_nDotRadius = max(3, _ttoi(sR));
    m_nThickness = max(1, _ttoi(sT));

    CRect canvas(0,0,700,650);
    pDC->FillSolidRect(canvas, RGB(245,247,252));
    pDC->SetBkMode(TRANSPARENT);

    CPen borderPen(PS_SOLID,1,RGB(180,180,200));
    CPen* pOldPen = pDC->SelectObject(&borderPen);
    pDC->SelectStockObject(NULL_BRUSH);
    pDC->Rectangle(canvas);
    pDC->SelectObject(pOldPen);

    // 선분
    if (m_nCount >= 2) {
        CPen sp(PS_SOLID,1,RGB(190,190,190));
        pDC->SelectObject(&sp);
        pDC->MoveTo(m_pts[0]); pDC->LineTo(m_pts[1]);
        if (m_nCount==3) { pDC->MoveTo(m_pts[1]); pDC->LineTo(m_pts[2]); }
    }

    // 수직이등분선
    auto drawPerp = [&](CPoint A, CPoint B, COLORREF col, LPCTSTR lbl) {
        double mx=(A.x+B.x)/2.0, my=(A.y+B.y)/2.0;
        double dx=B.x-A.x, dy=B.y-A.y, len=sqrt(dx*dx+dy*dy);
        if (len<1e-6) return;
        double px=-dy/len, py=dx/len;
        double ext=sqrt(700.0*700.0+650.0*650.0);
        CPen dp(PS_DASH,1,col); pDC->SelectObject(&dp);
        pDC->SelectStockObject(NULL_BRUSH);
        pDC->MoveTo((int)(mx-px*ext),(int)(my-py*ext));
        pDC->LineTo((int)(mx+px*ext),(int)(my+py*ext));
        int qs=6;
        POINT dm[5]={{(int)mx,(int)my-qs},{(int)mx+qs,(int)my},
                     {(int)mx,(int)my+qs},{(int)mx-qs,(int)my},{(int)mx,(int)my-qs}};
        CPen sp(PS_SOLID,1,col); pDC->SelectObject(&sp);
        pDC->Polyline(dm,5);
        double tx=dx/len,ty=dy/len; int q=7;
        POINT sq[5]={{(int)(mx+px*q),(int)(my+py*q)},
                     {(int)(mx+px*q+tx*q),(int)(my+py*q+ty*q)},
                     {(int)(mx+tx*q),(int)(my+ty*q)},
                     {(int)mx,(int)my},{(int)(mx+px*q),(int)(my+py*q)}};
        pDC->Polyline(sq,5);
        pDC->SetTextColor(col);
        CFont f; f.CreatePointFont(80,_T("맑은 고딕"));
        CFont* pf=pDC->SelectObject(&f);
        pDC->TextOut((int)(mx+px*40)+4,(int)(my+py*40)-14,lbl);
        pDC->SelectObject(pf);
    };
    if (m_nCount>=2) {
        drawPerp(m_pts[0],m_pts[1],RGB(220,120,0),_T("L₁"));
        if (m_nCount==3) drawPerp(m_pts[1],m_pts[2],RGB(140,0,200),_T("L₂"));
    }

    // 반지름선
    if (m_bCircleReady) {
        CPen rp(PS_DOT,1,RGB(120,160,255)); pDC->SelectObject(&rp);
        for (int i=0;i<3;i++) { pDC->MoveTo((int)m_cx,(int)m_cy); pDC->LineTo(m_pts[i]); }
        double mx2=(m_cx+m_pts[0].x)/2.0, my2=(m_cy+m_pts[0].y)/2.0;
        CString rl; rl.Format(_T("r=%.1f"),m_radius);
        pDC->SetTextColor(RGB(30,100,220));
        CFont rf; rf.CreatePointFont(80,_T("맑은 고딕"));
        CFont* prf=pDC->SelectObject(&rf);
        pDC->TextOut((int)mx2+3,(int)my2-13,rl);
        pDC->SelectObject(prf);
    }

    // 외접원 (두께 적용, Polyline)
    if (m_bCircleReady) {
        for (int t=0;t<m_nThickness;t++) {
            double off=t-m_nThickness/2.0;
            CPen cp(PS_SOLID,1,RGB(30,100,220)); pDC->SelectObject(&cp);
            pDC->SelectStockObject(NULL_BRUSH);
            DrawCircleByLine(pDC,m_cx,m_cy,m_radius+off);
        }
        CPen xp(PS_DOT,1,RGB(150,150,150)); pDC->SelectObject(&xp);
        int cx=(int)m_cx, cy=(int)m_cy;
        pDC->MoveTo(cx-18,cy); pDC->LineTo(cx+18,cy);
        pDC->MoveTo(cx,cy-18); pDC->LineTo(cx,cy+18);
        CPen cp2(PS_SOLID,1,RGB(30,100,220));
        pDC->SelectObject(&cp2);
        DrawFilledCircleByLine(pDC,m_cx,m_cy,4);
        CString cl; cl.Format(_T("O(%.1f,%.1f)"),m_cx,m_cy);
        pDC->SetTextColor(RGB(30,100,220));
        CFont cf; cf.CreatePointFont(85,_T("맑은 고딕"));
        CFont* pcf=pDC->SelectObject(&cf);
        pDC->TextOut(cx+7,cy+5,cl);
        pDC->SelectObject(pcf);
    }

    // 클릭 지점 원 (3개까지만)
    for (int i=0;i<min(m_nCount,3);i++) {
        int x=m_pts[i].x, y=m_pts[i].y;
        CPen dp(PS_SOLID,1,RGB(200,40,40)); pDC->SelectObject(&dp);
        DrawFilledCircleByLine(pDC,x,y,m_nDotRadius);
        CString lb; lb.Format(_T("P%d(%d,%d)"),i+1,x,y);
        pDC->SetTextColor(RGB(160,20,20));
        CFont pf; pf.CreatePointFont(88,_T("맑은 고딕"));
        CFont* ppf=pDC->SelectObject(&pf);
        pDC->TextOut(x+m_nDotRadius+3,y-14,lb);
        pDC->SelectObject(ppf);
    }

    // 상단 메시지
    CFont inf; inf.CreatePointFont(100,_T("맑은 고딕"));
    CFont* pif=pDC->SelectObject(&inf);
    if (m_bCannotDraw) {
        pDC->SetTextColor(RGB(200,30,30));
        pDC->TextOut(10,10,_T("⚠ 세 점이 일직선 — 원을 그릴 수 없습니다."));
    } else if (m_bCircleReady) {
        pDC->SetTextColor(RGB(30,100,220));
        CString s; s.Format(_T("✔ 외접원  O(%.1f,%.1f)  r=%.1f"),m_cx,m_cy,m_radius);
        pDC->TextOut(10,10,s);
    } else {
        pDC->SetTextColor(RGB(80,80,80));
        CString s; s.Format(_T("좌클릭으로 점을 찍으세요 (%d/3)"),m_nCount);
        pDC->TextOut(10,10,s);
    }
    pDC->SelectObject(pif);

    // 정보 패널 업데이트
    CString info;
    for (int i=0;i<m_nCount;i++)
        info.AppendFormat(_T("P%d:(%d,%d)\r\n"),i+1,m_pts[i].x,m_pts[i].y);
    if (m_bCircleReady)
        info.AppendFormat(_T("\r\nO(%.1f,%.1f)\r\nr=%.1f"),m_cx,m_cy,m_radius);
    else if (m_bCannotDraw) info+=_T("\r\n⚠ 일직선");
    m_staticInfo.SetWindowText(info);
}

// ── OnPaint ───────────────────────────────────────────
void CCircleFromPointsDlg::OnPaint()
{
    if (IsIconic()) {
        CPaintDC dc(this);
        SendMessage(WM_ICONERASEBKGND,
                    reinterpret_cast<WPARAM>(dc.GetSafeHdc()),0);
        int cx2=GetSystemMetrics(SM_CXICON)/2,cy2=GetSystemMetrics(SM_CYICON)/2;
        CRect r; GetClientRect(&r);
        dc.DrawIcon((r.Width()-cx2)/2,(r.Height()-cy2)/2,m_hIcon);
        return;
    }
    CPaintDC dc(this);
    CRect cr; GetClientRect(&cr);
    CDC memDC; memDC.CreateCompatibleDC(&dc);
    CBitmap bmp; bmp.CreateCompatibleBitmap(&dc,cr.Width(),cr.Height());
    CBitmap* pOld=memDC.SelectObject(&bmp);
    memDC.FillSolidRect(cr,::GetSysColor(COLOR_BTNFACE));
    DrawScene(&memDC);
    dc.BitBlt(0,0,cr.Width(),cr.Height(),&memDC,0,0,SRCCOPY);
    memDC.SelectObject(pOld);
}

// ── 마우스 ────────────────────────────────────────────
void CCircleFromPointsDlg::OnLButtonDown(UINT nFlags, CPoint point)
{
    if (point.x>=700) { CDialogEx::OnLButtonDown(nFlags,point); return; }
    if (m_nCount==3 && m_bCircleReady) {
        int idx=-1;
        if (HitTestPoint(point,idx)) {
            m_bDragging=true; m_nDragIdx=idx; SetCapture();
            CDialogEx::OnLButtonDown(nFlags,point); return;
        }
        m_nCount=0; m_bCircleReady=false; m_bCannotDraw=false;
    }
    if (m_nCount<3) {
        m_pts[m_nCount++]=point;
        if (m_nCount==3) CalcCircumCircle();
        Invalidate(FALSE);
    }
    CDialogEx::OnLButtonDown(nFlags,point);
}

void CCircleFromPointsDlg::OnLButtonUp(UINT nFlags, CPoint point)
{
    if (m_bDragging) { m_bDragging=false; m_nDragIdx=-1; ReleaseCapture(); }
    CDialogEx::OnLButtonUp(nFlags,point);
}

void CCircleFromPointsDlg::OnMouseMove(UINT nFlags, CPoint point)
{
    if (m_bDragging && m_nDragIdx>=0) {
        point.x=max(1,min(point.x,699));
        point.y=max(1,min(point.y,639));
        m_pts[m_nDragIdx]=point;
        CalcCircumCircle();
        Invalidate(FALSE);
    }
    CDialogEx::OnMouseMove(nFlags,point);
}

// ── 버튼 ──────────────────────────────────────────────
void CCircleFromPointsDlg::OnBtnReset()
{
    m_bThreadRun=false;
    if (m_pThread) { WaitForSingleObject(m_pThread->m_hThread,3000); m_pThread=nullptr; }
    m_nCount=0; m_bCircleReady=false; m_bCannotDraw=false;
    m_bDragging=false; m_nDragIdx=-1;
    Invalidate(FALSE);
}

void CCircleFromPointsDlg::OnBtnRandom()
{
    if (!m_bCircleReady) { AfxMessageBox(_T("먼저 세 점을 찍어 원을 완성하세요.")); return; }
    if (m_bThreadRun) return;
    m_bThreadRun=true;
    m_pThread=AfxBeginThread(RandomThreadProc,this,THREAD_PRIORITY_NORMAL,0,0,nullptr);
}

UINT CCircleFromPointsDlg::RandomThreadProc(LPVOID pParam)
{
    CCircleFromPointsDlg* p=reinterpret_cast<CCircleFromPointsDlg*>(pParam);
    for (int i=0;i<10&&p->m_bThreadRun;i++) {
        for (int j=0;j<3;j++) {
            p->m_pts[j].x=30+rand()%638;
            p->m_pts[j].y=30+rand()%578;
        }
        p->CalcCircumCircle();
        p->PostMessage(WM_REDRAW_REQUEST,0,0);
        Sleep(500);
    }
    p->m_bThreadRun=false;
    return 0;
}

LRESULT CCircleFromPointsDlg::OnRedrawRequest(WPARAM, LPARAM)
{
    Invalidate(FALSE); UpdateWindow(); return 0;
}
```

---

## 12. 빌드 및 실행

### 12.1 빌드 및 실행 단축키

| 단축키 | 동작 |
|--------|------|
| `Ctrl+Shift+B` | 빌드 (솔루션 빌드) |
| `Ctrl+F5` | 실행 (디버거 없이) |
| `F5` | 디버그 실행 |
| `Ctrl+Break` | 빌드 중지 |

### 12.2 MFC 워크로드 설치 확인 (VS 2026)

```
Visual Studio Installer 실행
→ Visual Studio 2026 → [수정]
→ 워크로드 탭: "C++를 사용한 데스크톱 개발" ✅
→ 개별 구성 요소 탭: "MSVC v14.x용 C++ MFC" ✅
→ [수정] 클릭
```

### 12.3 문자 집합 설정 (VS 2026)

```
솔루션 탐색기 → 프로젝트 우클릭 → 속성
→ 구성 속성 → 고급
→ 문자 집합: "유니코드 문자 집합 사용"
→ [확인]
```

> 🔄 **VS 2022와의 차이**  
> VS 2026에서는 속성 창도 Fluent UI로 재설계되어  
> 설정 항목 검색이 더 빠르고 정확해졌습니다.  
> 경로 자체(구성 속성 → 고급 → 문자 집합)는 동일합니다.

### 12.4 GitHub Copilot 활용 (VS 2026 신기능)

VS 2026은 GitHub Copilot이 IDE에 깊이 통합되어 있습니다.  
MFC 코드 작성 시 다음과 같이 활용할 수 있습니다:

```
- 인라인 코드 제안: 함수명 입력 시 자동 완성
- 채팅 패널:  Copilot에게 MFC 관련 질문 가능
- Cloud Agent: 반복적인 코드 리팩토링 위임 가능
```

---

## 13. 동작 요약

| 동작 | 결과 |
|------|------|
| 좌클릭 1~2번 | 빨간 점(Polyline 원) 표시, 수직이등분선 점진적 표시 |
| 좌클릭 3번 | 외접원 자동 계산 및 표시, 반지름선 표시 |
| 일직선 3점 | "⚠ 일직선" 경고 메시지 |
| 점 클릭 후 드래그 | 실시간으로 외접원 재계산 및 이동 |
| 4번째 이후 클릭 | 새 점 입력 시작 (점 원은 3개까지만) |
| [초기화] 클릭 | 전체 초기화 |
| [랜덤 이동] 클릭 | 별도 쓰레드로 10회×0.5초 간격 랜덤 이동 |
| 반지름/두께 입력 | 즉시 다음 그리기에 반영 |

---

## 14. 트러블슈팅

### Q1. `m_hIcon`: 선언되지 않은 식별자

```
원인: CDialogEx는 m_hIcon을 자동 제공하지 않음
해결: 헤더에 직접 선언
      HICON m_hIcon;
```

### Q2. `OnSysCommand` / `OnQueryDragIcon` 컴파일 오류

```
원인: 헤더에 afx_msg 선언 없음
해결: 헤더 protected 섹션에 추가
      afx_msg void OnSysCommand(UINT nID, LPARAM lParam);
      afx_msg HCURSOR OnQueryDragIcon();
```

### Q3. 화면 깜빡임

```
원인: OnPaint에서 직접 DC에 그리면 깜빡임 발생
해결: 메모리 DC에 먼저 그리고 BitBlt로 한 번에 복사
      (더블 버퍼링 패턴)
```

### Q4. 랜덤 이동 중 UI 프리징

```
원인: 작업 쓰레드에서 직접 Invalidate() 호출
해결: PostMessage(WM_REDRAW_REQUEST) 로 메인 쓰레드에 위임
```

### Q5. 드래그 중 창 밖으로 나가면 이벤트 끊김

```
원인: 마우스 캡처 없음
해결: OnLButtonDown에서 SetCapture()
      OnLButtonUp에서 ReleaseCapture()
```

### Q6. 원이 각져 보임

```
원인: DrawCircleByLine의 segments 값이 너무 작음
해결: 기본값 360 → 필요시 720으로 증가
      DrawCircleByLine(pDC, cx, cy, r, 720);
```

### Q7. VS 2026에서 MFC 템플릿이 보이지 않음

```
원인: "C++를 사용한 데스크톱 개발" 워크로드 미설치
해결:
  1. Visual Studio Installer 실행
  2. VS 2026 → [수정] 클릭
  3. 워크로드: "C++를 사용한 데스크톱 개발" 체크
  4. 개별 구성 요소: "MSVC v14.x용 C++ MFC" 체크
  5. [수정] → 설치 완료 후 재시작
```

### Q8. VS 2026에서 설정(Options) 항목을 못 찾겠음

```
원인: VS 2026은 설정 UI가 Fluent 기반으로 전면 재설계됨
      일부 설정이 새 위치로 이동했음

해결:
  - 새 설정 UI: 도구 → 옵션 → 검색창에 키워드 입력 (AI 검색 지원)
  - 아직 이전되지 않은 레거시 설정: "More Settings" 노드에서 접근
  - 예) 문자 집합 설정은 여전히 프로젝트 속성에서 접근
         솔루션 탐색기 → 프로젝트 우클릭 → 속성 → 구성 속성 → 고급
```

### Q9. VS 2026에서 리소스 뷰가 보이지 않음

```
해결:
  - 메뉴: 보기 → 리소스 뷰  (또는 Ctrl+Shift+E)
  - VS 2026은 메뉴 구조가 단순화되어
    "보기 → 다른 창 → 리소스 뷰" 경로가
    "보기 → 리소스 뷰" 로 단축되었을 수 있음
```

### Q10. 솔루션 탐색기 항목이 너무 넓게 표시됨

```
원인: VS 2026은 접근성 향상을 위해 기본 간격이 넓어짐
해결:
  도구 → 옵션 → 환경 → Visual Experience
  → "솔루션 탐색기에서 간격 줄이기" 체크
  → 즉시 적용됨 (재시작 불필요)
```

---

---

## 참고 자료

| 항목 | 링크 |
|------|------|
| VS 2026 공식 릴리즈 노트 | https://learn.microsoft.com/en-us/visualstudio/releases/2026/release-notes |
| VS 2026 새 UX 소개 | https://devblogs.microsoft.com/visualstudio/a-first-look-at-the-all-new-ux-in-visual-studio-2026/ |
| MFC 앱 만들기 공식 문서 | https://learn.microsoft.com/ko-kr/cpp/mfc/reference/creating-an-mfc-application |
| MFC 공식 클래스 레퍼런스 | https://learn.microsoft.com/ko-kr/cpp/mfc |
| AfxBeginThread | https://learn.microsoft.com/ko-kr/cpp/mfc/reference/cwinthread-class |
| CDC::Polyline | https://learn.microsoft.com/ko-kr/cpp/mfc/reference/cdc-class#polyline |
| SetCapture | https://learn.microsoft.com/ko-kr/windows/win32/api/winuser/nf-winuser-setcapture |
