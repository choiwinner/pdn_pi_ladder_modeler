# 📘 PDN 정밀 RLC 모델러 [모델 A] 사용자 가이드
## 모델 형태: 단일 버스 3-Branch 병렬 등가 모델 (Single-Bus 3-Branch Parallel Model)
**대상 파일**: [pdn_pi_ladder_modeler_a.html](file:///c:/python/Z_modeling/pdn_pi_ladder_modeler_a.html) (또는 [gemini-code-1790683335132.html](file:///c:/python/Z_modeling/gemini-code-1790683335132.html))

---

## 1. 모델 개요 및 이론적 배경

### 1.1 모델 A의 정의
**모델 A**는 VRM 전원 경로의 순수 직렬 DC 저항($R_{DC}$) 뒤에, 서로 다른 주파수 응답을 갖는 **3개의 R-L-C 직렬 브랜치가 단일 버스(PDN_Bus)에 병렬(Shunt)**로 매달려 있는 구조입니다. 모든 고주파 기생 성분이 커패시터 브랜치 내부의 실장 ESL($L_{m1}, L_{m2}$)로 등가 흡수되어 간결하고 명확하게 전체 임피던스 프로파일을 추종합니다.

```
[PORT 1: VRM]
     │
   [R_dc] (5.69mΩ, 10A 인가 시 56.9mV DC 전압 강하)
     │
   Node_PDN ─────────────────────────────────────────────────────────────> [PORT 2: DUT Die]
     │                                │                                │
[Branch 1: 탄탈 뱅크]            [Branch 2: 33uF MLCC군]          [Branch 3: 10u/1u MLCC군]
  R_bulk (선반 ESR, 10mΩ)          R_m1 (최저 바닥 저항, 1.0mΩ)     R_m2 (2차 바닥 저항, 2.0mΩ)
  C_bulk (벌크 용량, 9,400uF)      C_m1 (33u 실효 용량, ~800uF)     C_m2 (10u/1u 용량, ~30uF)
  L_bulk (탄탈 ESL, ~20nH)         L_m1 (33u 실장 ESL, ~150pH)      L_m2 (10u/1u ESL, ~60pH)
     │                                │                                │
    GND                              GND                              GND
```

---

### 1.2 수학적 전달함수 (Complex Impedance Formulation)

PORT 2(DUT Die)에서 바라본 복소 구동점 임피던스 $Z(f)$는 세 브랜치 어드미턴스(Admittance)의 병렬 합으로 표현됩니다:

$$Z_{k}(f) = R_{k} + j \left( 2\pi f L_{k} - \frac{1}{2\pi f C_{k}} \right) \quad (k \in \{bulk, m1, m2\})$$

$$Y_{total}(f) = \frac{1}{Z_{bulk}(f)} + \frac{1}{Z_{m1}(f)} + \frac{1}{Z_{m2}(f)}$$

$$Z_{PORT2}(f) = \frac{1}{Y_{total}(f)}$$

$$|Z_{PORT2}(f)| = \sqrt{\left(\operatorname{Re}(Z_{PORT2})\right)^2 + \left(\operatorname{Im}(Z_{PORT2})\right)^2}$$

#### 주파수 대역별 거동 메커니즘
1. **$10\text{ Hz} \sim 1\text{ kHz}$ (용량성 하강)**: $C_{bulk} \approx 9,400\text{ }\mu\text{F}$가 지배하여 $|Z| \approx \frac{1}{2\pi f C_{bulk}}$로 $-20\text{ dB/dec}$ 급강하.
2. **$1\text{ kHz} \sim 10\text{ kHz}$ (탄탈 ESR 선반)**: $C_{bulk}$의 리액턴스가 매우 작아져 $R_{bulk} \approx 10\text{ m}\Omega$의 평탄한 저항성 선반(Shelf) 형성.
3. **$10\text{ kHz} \sim 680\text{ kHz}$ (1차 주공진)**: 대용량 MLCC 33µF 뱅크($C_{m1}$)가 임피던스를 다시 끌어내려 $f_{v1} = \frac{1}{2\pi \sqrt{L_{m1} C_{m1}}} \approx 680\text{ kHz}$에서 최저점 $|Z| = R_{m1} \approx 1.0\text{ m}\Omega$ 형성.
4. **$1\text{ MHz} \sim 10\text{ MHz}$ (반공진 피크 & 2차 공진)**: $L_{m1}$과 $C_{m2}$의 병렬 공진으로 $2.5\text{ MHz}$ 피크 형성 후, $C_{m2}$와 $L_{m2}$의 직렬 공진으로 $5.5\text{ MHz}$ 2차 바닥($2.0\text{ m}\Omega$) 형성.
5. **$10\text{ MHz} \sim 1\text{ GHz}$ (고주파 인덕티브 상승)**: 모든 커패시터가 쇼트되고 $L_{parallel} = L_{m1} \parallel L_{m2}$가 남아 $+20\text{ dB/dec}$로 지속 상승.

---

## 2. ADS 시뮬레이션 설정 및 데이터 추출 가이드

### 2.1 DC IR Drop 시뮬레이션 및 $R_{DC}$ 확인법

```
[DC Simulation Setup in ADS]
+-----------------+       +-------------------+       +-----------------+
| V_DC Source     |───────| S-parameter Block |───────| I_Probe /       |
| (1.2V, Ideal)   | Port 1| (EM / SIwave 모델)| Port 2| Current Sink    |
+-----------------+       +-------------------+       | (10A Step/DC)   |
        │                                             +-----------------+
       GND                                                     │
                                                              GND
```

1. **ADS 회로 구성**:
   - `Term1 (Port 1)`에 DC 전압원(예: 1.2V)을 연결합니다.
   - `Term2 (Port 2: DUT 칩셋 패드)`에 **I_DC(10.0A)** 전류원(또는 Current Sink)을 연결합니다.
2. **시뮬레이션 실행**:
   - `DC Simulation Controller`를 배치하고 시뮬레이션을 실행합니다.
3. **결과 확인**:
   - Port 1 전압($V_1$)과 Port 2 전압($V_2$)의 차이 $\Delta V = V_1 - V_2$를 계산합니다.
   - **실제 계측값 예시**: $V_1 = 1.200\text{ V}, V_2 = 1.1431\text{ V} \implies \Delta V = 56.9\text{ mV}$
   - 이때 직산 저항:
     $$R_{DC} = \frac{\Delta V}{I_{sink}} = \frac{56.9\text{ mV}}{10.0\text{ A}} = 5.69\text{ m}\Omega$$
4. **HTML 입력**:
   - [gemini-code-1790683335132.html](file:///c:/python/Z_modeling/gemini-code-1790683335132.html)의 **1번 카드**에 `Isink = 10.0 A`, `ΔVdrop = 56.9 mV`를 입력합니다.

---

### 2.2 S-parameter AC 임피던스 시뮬레이션 및 마커 추출법

1. **ADS S-파라미터 컨트롤러 설정**:
   - Type: **Log**
   - Start Frequency: **10 Hz** (또는 최소 추출 주파수)
   - Stop Frequency: **1 GHz**
   - Points/Decade: **50 pts/dec 이상** 권장.
2. **Z-파라미터 변환 수식 (Data Display 창)**:
   - ADS Data Display 창에서 수식(Equation)을 작성합니다:
     ```text
     Z_mat = stoz(S, 50)
     Z22 = Z_mat(2,2)
     Z_mag = mag(Z22)
     ```
   - X축을 `Log Frequency(Hz)`, Y축을 `Log Z_mag(Ohms)`로 설정하여 직교 좌표 플롯을 생성합니다.

3. **마커(Marker) 5개 지점 측정 및 HTML 입력값 매핑**:

| 번호 | 마커 명칭 | ADS 주파수 위치 ($f$) | ADS 관측 임피던스 ($|Z|$) | 물리적 의미 |
| :---: | :--- | :--- | :--- | :--- |
| **①** | **저주파 용량성 하강** | `100 Hz` | `0.160 Ω` (160 mΩ) | 탄탈 470µF×20개 합성 용량 ($C_{bulk}$) 결정 |
| **②** | **탄탈 ESR 선반** | `5 kHz` (1k~10k 플랫 구간) | `0.010 Ω` (10 mΩ) | 탄탈 뱅크 합성 ESR 바닥 ($R_{bulk}$) 결정 |
| **③** | **1차 하강 슬로프** | `100 kHz` | `0.002 Ω` (2.0 mΩ) | 33µF MLCC 뱅크 실효 용량 ($C_{m1}$) 결정 |
| **③** | **1차 주공진 최저 바닥** | `680 kHz` (또는 600~700k 최저점) | `0.001 Ω` (1.0 mΩ) | 33µF MLCC 뱅크 실장 ESL & ESR 결정 |
| **④** | **반공진 피크 (Peak)** | `2.5 MHz` (솟아오른 험프) | `0.0025 Ω` (2.5 mΩ) | 33uF L과 10uF C 사이의 병렬 반공진점 |
| **④** | **2차 공진 최저 바닥** | `5.5 MHz` (두 번째 골짜기) | `0.0020 Ω` (2.0 mΩ) | 10µF/1µF 소형 MLCC 뱅크 공진점 |
| **⑤** | **고주파 인덕턴스 수렴** | `100 MHz` | `0.050 Ω` (50 mΩ) | 10MHz~1GHz 치솟는 $+20\text{dB/dec}$ 상승 기울기 |

---

## 3. 웹 모델러 실행 및 결과 활용

1. 브라우저에서 [gemini-code-1790683335132.html](file:///c:/python/Z_modeling/gemini-code-1790683335132.html)을 실행합니다.
2. 상기 5개 마커 지점의 주파수와 임피던스를 입력한 후 **[🚀 전체 파라미터 정밀 피팅 및 갱신]** 버튼을 누릅니다.
3. **정합 오차율 확인**:
   - 우측 하단 **마커 오차율 분석 테이블**에서 6~7개 마커의 오차율이 모두 **0.01% 미만(녹색)**으로 수렴하는지 확인합니다.
4. **산출된 Netlist 복사**:
   - 좌측 하단 5번 카드의 `.subckt PDN_RLC_MODEL` 넷리스트를 복사하여 LTspice 또는 ADS Schematic(SPICE Netlist Component)에 삽입합니다.

---

## 4. 모델 A Netlist 예시 (LTspice / ADS 호환)

```spice
* ========================================================
* LTspice Deterministic PDN Multi-Branch Equivalent Model
* Model A: Single-Bus 3-Branch Parallel Topology
* IR Drop: 56.9mV @ 10.0A (R_dc = 5.69mΩ)
* ========================================================
.subckt PDN_RLC_MODEL PORT_VRM PORT_DUT GND
* 1. DC Path Resistance (Power Plane & Contact Loop)
R_dc      PORT_VRM    Node_PDN     5.6900e-03

* 2. Bulk Tantalum Capacitor Bank (470uF x 20)
R_bulk    Node_PDN    n_b1         1.0120e-02
C_bulk    n_b1        n_b2         9.5970e-03
L_bulk    n_b2        GND          4.3300e-08

* 3. MLCC 33uF Bank (Valley 1 Shaper)
R_m1      Node_PDN    n_m1a        9.9000e-04
C_m1      n_m1a       n_m1b        3.4190e-04
L_m1      n_m1b       GND          1.0450e-10

* 4. MLCC 10uF / 1uF Bank (Peak & Valley 2 Shaper)
R_m2      Node_PDN    n_m2a        9.6000e-04
C_m2      n_m2a       n_m2b        2.5300e-05
L_m2      n_m2b       GND          2.3400e-11

* Direct connection to DUT Pad
W_dut     Node_PDN    PORT_DUT     0
.ends PDN_RLC_MODEL
```
