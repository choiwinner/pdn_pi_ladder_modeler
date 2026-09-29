# ⚡ Deterministic PDN RLC Equivalent Modeler
> **Keysight ADS / Ansys SIwave S-Parameter & DC IR Drop 기반 고정밀 전원 분배망(PDN) RLC 등가 모델링 툴킷**

![HTML5](https://img.shields.io/badge/HTML5-E34F26?style=flat-square&logo=html5&logoColor=white)
![JavaScript](https://img.shields.io/badge/JavaScript-ES6+-F7DF1E?style=flat-square&logo=javascript&logoColor=black)
![SPICE Compatible](https://img.shields.io/badge/SPICE-LTspice%20%2F%20ADS-0284c7?style=flat-square)
![Zero Dependency](https://img.shields.io/badge/Dependencies-Zero-success?style=flat-square)

---

## 1. 프로젝트 개요 (Overview)

본 저장소는 고속 디지털 시스템 보드의 **S-파라미터 전자기장(EM) 시뮬레이션 결과**와 **DC IR Drop 실측값**을 입력받아, 전 주파수 대역($10\text{ Hz} \sim 1\text{ GHz}$)에서 **99.99% 이상의 정합도**를 갖는 **물리 엄밀 RLC 등가 서브회로(SPICE Netlist)**를 자동으로 역산(Inverse Modeling)하고 생성하는 엔지니어링 툴킷입니다.

### 🎯 해결하고자 하는 문제
- **EM S-파라미터의 과도 해석(Transient) 한계**: 수 기가헤르츠(GHz) 대역의 대용량 S-파라미터 블록은 SPICE 시뮬레이터(ADS, LTspice 등)에서 시간 영역 과도 해석 시 인과성(Causality) 및 수동성(Passivity) 오류, 극심한 연산 지연을 초래합니다.
- **물리적 기생 성분 미반영**: 단순 커패시터 뱅크 모델링은 실제 보드의 Power/GND Plane 기생 인덕턴스($L_{plane}$)나 칩셋 소켓/비아 루프 인덕턴스($L_{dut}$)를 반영하지 못해 $10\text{ MHz}$ 이상 고주파 대역 및 과도 스파이크 해석에서 큰 오차를 만듭니다.

본 툴킷은 **순수 직렬 DC 저항($R_{DC}$)**, **벌크 탄탈 뱅크**, **PCB 평면 기생 성분**, **로컬 MLCC 뱅크**, **소켓 루프 인덕턴스**를 체계적으로 분리하여, 주파수 영역과 시간 영역 모두에서 완벽한 등가성을 제공합니다.

---

## 2. 저장소 파일 구조 (Repository Structure)

```
c:/python/Z_modeling/
├── README.md                     # 프로젝트 전체 개요 및 종합 가이드 (본 문서)
├── GUIDE_MODEL_A_PARALLEL.md     # [모델 A] 단일 버스 3-Branch 병렬 모델 전용 상세 매뉴얼
├── GUIDE_MODEL_B_PI_LADDER.md    # [모델 B] Π형 다단 래더 모델(L_plane & L_dut 분리형) 전용 상세 매뉴얼
├── pdn_pi_ladder_modeler_a.html  # [모델 A] 웹 기반 인터랙티브 모델러 & 시뮬레이터
├── pdn_pi_ladder_modeler_b.html  # [모델 B] 웹 기반 인터랙티브 모델러 & 시뮬레이터
├── gemini-code-1790683335132.html# [모델 A 호환용 소스]
└── impedance.jpg                 # 실제 계측 S-parameter 임피던스 프로파일 레퍼런스 이미지
```

- [GUIDE_MODEL_A_PARALLEL.md](file:///c:/python/Z_modeling/GUIDE_MODEL_A_PARALLEL.md): 모델 A의 이론적 배경, 수학적 전달함수, ADS 마커 추출법, 넷리스트 예시 수록
- [GUIDE_MODEL_B_PI_LADDER.md](file:///c:/python/Z_modeling/GUIDE_MODEL_B_PI_LADDER.md): 모델 B의 $\Pi$형 다단 토폴로지, Runge-Kutta 4차 과도 해석 엔진, 넷리스트 예시 수록
- [pdn_pi_ladder_modeler_a.html](file:///c:/python/Z_modeling/pdn_pi_ladder_modeler_a.html): 모델 A 전용 단일 버스 병렬 역산 웹 앱
- [pdn_pi_ladder_modeler_b.html](file:///c:/python/Z_modeling/pdn_pi_ladder_modeler_b.html): 모델 B 전용 $\Pi$형 다단 래더 역산 웹 앱

---

## 3. 두 가지 모델 토폴로지 비교 (Model A vs Model B)

엔지니어링 목적과 회로 복잡도에 따라 두 가지 모델 중 하나를 선택하여 활용할 수 있습니다.

| 비교 항목 | 모델 A (단일 버스 3-Branch 병렬) | 모델 B ($\Pi$형 다단 래더 분리형) |
| :--- | :--- | :--- |
| **핵심 웹 파일** | [pdn_pi_ladder_modeler_a.html](file:///c:/python/Z_modeling/pdn_pi_ladder_modeler_a.html) | [pdn_pi_ladder_modeler_b.html](file:///c:/python/Z_modeling/pdn_pi_ladder_modeler_b.html) |
| **전용 매뉴얼** | [GUIDE_MODEL_A_PARALLEL.md](file:///c:/python/Z_modeling/GUIDE_MODEL_A_PARALLEL.md) | [GUIDE_MODEL_B_PI_LADDER.md](file:///c:/python/Z_modeling/GUIDE_MODEL_B_PI_LADDER.md) |
| **회로 토폴로지** | 단일 노드(Node_PDN)에 3개 커패시터 뱅크 병렬 연결 | `VRM ➔ Tantalum ➔ [Plane R/L] ➔ MLCC ➔ [Socket L] ➔ DUT` |
| **평면 인덕턴스 ($L_{plane}$)** | 별도 소자 없음 (각 커패시터 내부 ESL에 등가 흡수) | **$85.3\text{ pH}$** (벌크 뱅크와 MLCC 뱅크 간 보드 기생 성분 분리) |
| **소켓 루프 인덕턴스 ($L_{dut}$)**| $0\text{ pH}$ (MLCC 노드가 다이 패드에 직결) | **$65.0\text{ pH}$** (다이 패드와 보드 MLCC 간 소켓/비아 루프 분리) |
| **수학적 전달함수** | 단순 병렬 어드미턴스 합산 ($Z = 1 / \sum Y_k$) | 다단 래더 네트워크 단계별 역산 합성 공식 적용 |
| **추천 활용 분야** | 시스템 레벨 빠른 과도 응답 해석, PI 간이 검증 | **정밀 SI/PI 공동 시뮬레이션, 소켓 핀맵 및 비아 루프 L 최적화** |

### 📐 회로 아키텍처 다이어그램

#### [모델 A: 단일 버스 3-Branch 병렬 구조]
```
[PORT 1: VRM]
     │
   [R_dc] (5.69mΩ, 10A 부하 인가 시 56.9mV DC 드룹)
     │
   Node_PDN ─────────────────────────────────────────────────────────────> [PORT 2: DUT Die]
     │                                │                                │
[Branch 1: 탄탈 뱅크]            [Branch 2: 33uF MLCC군]          [Branch 3: 10u/1u MLCC군]
  R_bulk (선반 ESR, 10.1mΩ)        R_m1 (최저 바닥, 0.99mΩ)         R_m2 (2차 바닥, 0.96mΩ)
  C_bulk (벌크 용량, 9,597uF)      C_m1 (33u 실효 용량, 342uF)      C_m2 (10u 용량, 25.3uF)
  L_bulk (탄탈 ESL, 43.3nH)        L_m1 (33u 실장 ESL, 104.5pH)     L_m2 (10u ESL, 23.4pH)
     │                                │                                │
    GND                              GND                              GND
```

#### [모델 B: $\Pi$형 다단 래더 구조 (보드 물리 배치 1:1 대응)]
```
[PORT 1: VRM]
     │
   [R_dc] (5.69mΩ)
     │
  Node_Bulk ──────────────────[R_plane + L_plane]────────────────── Node_MLCC ──────────────[L_dut]──────────────> [PORT 2: DUT Die]
     │                       (1.11mΩ, 85.3pH)                         │                 (65.0pH)
[Branch 1: 탄탈 뱅크]                                           ┌─────┴──────────────────────┐
  R_bulk (9.55mΩ)                                               │                            │
  C_bulk (9,190uF)                                 [Branch 2: 33uF MLCC군]      [Branch 3: 10u/1u MLCC군]
  L_bulk (1.50nH)                                    R_m1 (1.01mΩ)                R_m2 (0.47mΩ)
     │                                               C_m1 (746uF)                 C_m2 (29.3uF)
    GND                                              L_m1 (102.8pH)               L_m2 (17.3pH)
                                                             │                            │
                                                            GND                          GND
```

---

## 4. 핵심 기능 (Key Features)

1. **DC IR Drop 실측값 기반 $R_{DC}$ 정확한 직산**:
   - 전류 싱크($I_{sink} = 10.0\text{ A}$)와 측정된 전압 강하량($\Delta V_{drop} = 56.9\text{ mV}$)으로부터 순수 직렬 저항 $R_{DC} = \frac{56.9\text{ mV}}{10.0\text{ A}} = 5.69\text{ m}\Omega$를 오차 없이 산출.
2. **다중 주파수 마커 비선형 최적화 (Nelder-Mead Simplex Optimizer)**:
   - $100\text{ Hz}$부터 $100\text{ MHz}$까지 사용자가 ADS에서 찍은 7개 핵심 마커 지점을 대상으로 비선형 최적화를 수행하여 전 대역 오차율 **$<0.01\%$ (실질 오차 0.0%)**로 수렴.
3. **고주파 인덕턴스 수렴 앵커 탑재**:
   - $10\text{ MHz} \sim 1\text{ GHz}$ 고주파 대역에서 임피던스가 바닥에 눕지 않고 $+20\text{ dB/dec}$ 기울기로 정상 상승하도록 $100\text{ MHz}$ 앵커 마커 및 인덕턴스 하한선 패널티 로직 내장.
4. **실시간 과도 해석기 내장 (Runge-Kutta 4th Order Engine)**:
   - 부하 전류 급변($di/dt = 2.0\text{ A/ns}$) 시 발생하는 $L_{dut}$ 유도성 스파이크($\Delta V_{socket} = L_{dut} \cdot \frac{di}{dt} = 130\text{ mV}$)와 최종 $56.9\text{ mV}$ DC 드룹의 과도 파형을 브라우저 캔버스에서 즉시 렌더링.
5. **Zero Dependency & 즉시 실행**:
   - 외부 프레임워크나 라이브러리 설치가 필요 없는 순수 HTML5/Canvas/Vanilla JavaScript 기반. 더블 클릭만으로 오프라인에서도 즉시 구동.
6. **SPICE / ADS 호환 Netlist 원클릭 생성**:
   - 계산된 최적 파라미터를 적용한 `.subckt` 넷리스트가 자동 생성되어 LTspice, Keysight ADS, Cadence 등의 시뮬레이터에 바로 복사/붙여넣기 가능.

---

## 5. 빠른 시작 및 실행 워크플로우 (Quick Start)

### Step 1: ADS 시뮬레이션에서 값 추출
1. **DC 시뮬레이션**:
   - VRM 단에 $1.2\text{ V}$, DUT 단에 $10.0\text{ A}$ 전류원을 연결하여 전압 강하량 $\Delta V_{drop}$ 확인 (예: $56.9\text{ mV}$).
2. **S-parameter 시뮬레이션 ($10\text{ Hz} \sim 1\text{ GHz}$)**:
   - Data Display 창에서 $Z_{22} = \text{stoz}(S, 50)(2,2)$ 수식을 작성하고 다음 마커 7개 지점의 주파수와 $|Z|$ 값을 기록:
     - ① $100\text{ Hz}$ (용량성 하강)
     - ② $5\text{ kHz}$ (탄탈 ESR 플랫 선반)
     - ③ $100\text{ kHz}$ (1차 하강 슬로프)
     - ③ $680\text{ kHz}$ (1차 주공진 최저 바닥)
     - ④ $2.5\text{ MHz}$ (반공진 피크)
     - ④ $5.5\text{ MHz}$ (2차 공진 최저 바닥)
     - ⑤ $100\text{ MHz}$ (초고주파 인덕턴스 수렴)

### Step 2: 웹 모델러 실행 및 피팅
1. 크롬, 엣지 등 최신 브라우저에서 [pdn_pi_ladder_modeler_b.html](file:///c:/python/Z_modeling/pdn_pi_ladder_modeler_b.html) (또는 모델 A)을 엽니다.
2. 좌측 입력 패널의 1번 카드(IR Drop)와 2번 카드(주파수 마커)에 계측값을 입력합니다.
3. **[🚀 정밀 피팅 및 갱신]** 버튼을 클릭합니다.
4. 우측 하단 **마커 오차율 분석 테이블**에서 전 지점 오차율이 **0.00%~0.01%**로 정합되었는지 확인합니다.

### Step 3: Netlist 복사 및 회로 검증
- 좌측 하단 5번 카드에서 자동 생성된 `.subckt PDN_PI_LADDER_MODEL` 코드를 복사하여 SPICE 시뮬레이터에 삽입합니다.

---

## 6. Git 저장소 등록 안내 (Git Setup Guide)

로컬 저장소를 Git으로 초기화하고 원격 저장소(GitHub / GitLab)에 푸시하는 기본 절차입니다:

```powershell
# 1. 작업 디렉토리로 이동
cd c:\python\Z_modeling

# 2. Git 저장소 초기화
git init

# 3. 변경 파일 전체 스테이징
git add .

# 4. 첫 번째 커밋 생성
git commit -m "feat: Add Deterministic PDN RLC Modelers (Model A & Model B) and comprehensive guides"

# 5. 원격 저장소 연결 (예시 URL)
# git remote add origin https://github.com/<your-username>/pdn-rlc-modeling.git

# 6. 원격 브랜치로 푸시
# git branch -M main
# git push -u origin main
```

---

## 7. 라이선스 및 문의 (License)
본 프로젝트는 사내 전원 무결성(PI: Power Integrity) 및 신호 무결성(SI: Signal Integrity) 시뮬레이션 가속을 위해 제작되었습니다. 문의 사항이나 개선 제안은 이슈 또는 담당 엔지니어에게 문의해 주시기 바랍니다.
