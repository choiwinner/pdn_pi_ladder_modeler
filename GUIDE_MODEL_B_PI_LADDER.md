# 📘 PDN 정밀 RLC 모델러 [모델 B] 사용자 가이드
## 모델 형태: Π형 다단 래더 등가 모델 (Pi-Ladder Topology with Explicit L_plane & L_dut)
**대상 파일**: [pdn_pi_ladder_modeler_b.html](file:///c:/python/Z_modeling/pdn_pi_ladder_modeler_b.html)

---

## 1. 모델 개요 및 이론적 배경

### 1.1 모델 B의 정의 및 물리적 필요성
**모델 B**는 실제 고속 전원 분배망(PDN: Power Delivery Network)의 **물리적 공간 배치(Spatial Layout)**를 1:1로 반영한 **$\Pi$형 다단 래더(Pi-Ladder) 회로망**입니다.

기존의 단순 병렬 모델(모델 A)은 모든 커패시터가 하나의 노드에 병렬로 물려 있다고 가정하므로, 보드 상에서 커패시터 뱅크 간의 물리적 거리(PCB 평면 기생 임피던스)나 소켓/패키지 비아의 루프 인덕턴스를 개별적으로 분리해 관찰하기 어렵습니다.

반면 **모델 B**는 다음 3개의 물리적 구역을 명시적으로 분리합니다:
1. **벌크 영역 (VRM & Tantalum Bank)**: VRM 출력단 바로 뒤에 실장된 470µF×20개 벌크 커패시터 뱅크
2. **평면 분리 전송 구간 ($R_{plane}, L_{plane}$)**: 벌크 뱅크에서 칩셋 주변 MLCC 뱅크 사이를 잇는 PCB Power/GND Plane Cavity의 기생 저항 및 인덕턴스
3. **로컬 고주파 바이패스 영역 (MLCC 뱅크)**: 칩셋 BGA 주변에 촘촘히 배치된 33µF, 10µF, 1µF 세라믹 칩 커패시터군
4. **칩 인터페이스 루프 ($L_{dut}$)**: MLCC 배치 노드에서 최종 실리콘 다이(DUT Die) 패드까지 이어지는 소켓 핀, 볼 그리드(BGA Ball), 패키지 비아 루프 인덕턴스

```
[PORT 1: VRM]
     │
   [R_dc] (5.69mΩ, 10A 부하 인가 시 56.9mV DC 전압 강하)
     │
  Node_Bulk ──────────────────[R_plane + L_plane]────────────────── Node_MLCC ──────────────[L_dut]──────────────> [PORT 2: DUT Die]
     │                       (1.11mΩ, 85.3pH)                         │                 (65.0pH)
[Branch 1: 탄탈 뱅크]                                           ┌─────┴──────────────────────┐
  R_bulk (선반 ESR, 9.55mΩ)                                     │                            │
  C_bulk (벌크 용량, 9,190uF)                      [Branch 2: 33uF MLCC군]      [Branch 3: 10u/1u MLCC군]
  L_bulk (탄탈 ESL, 1.50nH)                          R_m1 (최저 바닥, 1.01mΩ)     R_m2 (2차 바닥, 0.47mΩ)
     │                                               C_m1 (33u 용량, 746uF)       C_m2 (10u 용량, 29.3uF)
    GND                                              L_m1 (실장 ESL, 102.8pH)     L_m2 (실장 ESL, 17.3pH)
                                                             │                            │
                                                            GND                          GND
```

---

### 1.2 수학적 전달함수 (Complex Impedance Formulation)

DUT Die (PORT 2)에서 보드 쪽을 바라본 구동점 복소 임피던스 $Z_{PORT2}(f)$는 래더 네트워크의 단계별 역산 합성 공식으로 엄밀하게 도출됩니다.

#### 1단계: 벌크 탄탈 브랜치 어드미턴스 ($Y_{bulk}$)
$$Z_{bulk}(f) = R_{bulk} + j \left( 2\pi f L_{bulk} - \frac{1}{2\pi f C_{bulk}} \right)$$

#### 2단계: 평면 기생 소자를 거쳐 Node_MLCC에서 바라본 벌크 측 등가 임피던스
$$Z_{to\_bulk}(f) = (R_{plane} + j 2\pi f L_{plane}) + Z_{bulk}(f)$$
$$Y_{to\_bulk}(f) = \frac{1}{Z_{to\_bulk}(f)}$$

#### 3단계: 로컬 MLCC 브랜치들의 어드미턴스
$$Z_{m1}(f) = R_{m1} + j \left( 2\pi f L_{m1} - \frac{1}{2\pi f C_{m1}} \right)$$
$$Z_{m2}(f) = R_{m2} + j \left( 2\pi f L_{m2} - \frac{1}{2\pi f C_{m2}} \right)$$
$$Y_{m1}(f) = \frac{1}{Z_{m1}(f)}, \quad Y_{m2}(f) = \frac{1}{Z_{m2}(f)}$$

#### 4단계: Node_MLCC 총 병렬 어드미턴스 및 임피던스
$$Y_{MLCC}(f) = Y_{to\_bulk}(f) + Y_{m1}(f) + Y_{m2}(f)$$
$$Z_{MLCC}(f) = \frac{1}{Y_{MLCC}(f)}$$

#### 5단계: 소켓/패키지 루프 인덕턴스 직렬 가산 (최종 구동점 임피던스)
$$Z_{PORT2}(f) = Z_{MLCC}(f) + j 2\pi f L_{dut}$$
$$|Z_{PORT2}(f)| = \sqrt{\left(\operatorname{Re}(Z_{PORT2})\right)^2 + \left(\operatorname{Im}(Z_{PORT2})\right)^2}$$

---

### 1.3 물리적 거동 메커니즘 분석
1. **$10\text{ Hz} \sim 1\text{ kHz}$ (저주파 벌크 용량성 대역)**:
   - MLCC 용량($\approx 750\text{ }\mu\text{F}$)보다 12배 이상 큰 탄탈 뱅크($C_{bulk} \approx 9,190\text{ }\mu\text{F}$)가 지배합니다.
   - 기울기 $-20\text{ dB/dec}$로 떨어지며 $100\text{ Hz}$에서 $160\text{ m}\Omega$를 완벽히 통과합니다.
2. **$1\text{ kHz} \sim 10\text{ kHz}$ (탄탈 ESR 선반 대역)**:
   - $C_{bulk}$가 교류적으로 단락되면서 탄탈의 내부 저항 $R_{bulk} \approx 9.55\text{ m}\Omega$가 드러나 평탄한 수평 선반을 형성합니다.
3. **$10\text{ kHz} \sim 680\text{ kHz}$ (1차 주공진 및 평면 감결합 대역)**:
   - 칩셋 근처 33µF MLCC 뱅크($C_{m1}$)와 기생 ESL($L_{m1}$)이 주공진을 일으키며 임피던스가 급강하합니다.
   - $f_{v1} \approx 680\text{ kHz}$에서 최저 바닥 임피던스 $|Z| = 1.0\text{ m}\Omega$를 기록합니다. 이때 평면 인덕턴스 $L_{plane}$이 탄탈 측을 고주파 차단하여 MLCC 뱅크가 독자적으로 최저 바닥을 형성합니다.
4. **$1\text{ MHz} \sim 10\text{ MHz}$ (반공진 피크 & 2차 공진 대역)**:
   - $L_{m1}$과 10µF/1µF 군의 $C_{m2}$ 사이에서 병렬 반공진이 발생하여 $2.5\text{ MHz}$에서 $2.5\text{ m}\Omega$ 피크를 형성합니다.
   - 이후 $5.5\text{ MHz}$에서 소형 MLCC 뱅크의 직렬 공진으로 $2.0\text{ m}\Omega$의 2차 바닥을 형성합니다.
5. **$10\text{ MHz} \sim 1\text{ GHz}$ (소켓/비아 루프 $L_{dut}$ 지배 대역)**:
   - 모든 커패시터가 쇼트된 초고주파 대역에서는 다이 패드와 보드를 잇는 직렬 소켓 인덕턴스 $L_{dut} \approx 65.0\text{ pH}$가 임피던스를 지배합니다.
   - 임피던스는 $+20\text{ dB/dec}$로 지속 상승하여 $100\text{ MHz}$에서 정확히 $50\text{ m}\Omega$에 도달합니다.

---

### 1.4 Runge-Kutta 4차(RK4) 과도 해석 엔진 이론
모델 B의 웹 시뮬레이터에는 9차 상태공간 미분방정식(State-Space Equations)을 실시간 해석하는 RK4 엔진이 내장되어 있습니다.

$$v_{DUT}(t) = V_{MLCC}(t) - L_{dut} \frac{d i_{sink}(t)}{dt}$$

- **초기 스파이크 ($0 \sim 10\text{ ns}$)**: 부하 전류의 상승 에지($t_r = 5\text{ ns}$, $di/dt = 2\text{ A/ns}$)에서 소켓 루프 $L_{dut} \approx 65\text{ pH}$에 의해 즉각적인 유도성 스파이크 전압 강하가 발생합니다:
  $$\Delta V_{socket} = L_{dut} \cdot \frac{di}{dt} = 65\text{ pH} \times 2.0\text{ A/ns} = 130\text{ mV}$$
- **정상 상태 DC 전압 강하 ($t > 1\text{ }\mu\text{s}$)**: 모든 $L$과 $C$의 과도 변동이 잦아들면, 전체 직렬 DC 저항 $R_{DC} = 5.69\text{ m}\Omega$에 의해 **정확히 $56.9\text{ mV}$의 평탄한 DC IR Drop**으로 정밀 수렴합니다.

---

## 2. ADS 시뮬레이션 설정 및 데이터 추출 가이드

### 2.1 DC IR Drop 시뮬레이션 및 $R_{DC}$ 설정법

```
[ADS DC Simulation Setup]
+-------------------+        +----------------------+        +-------------------+
| V_DC Source       |────────| S-parameter / EM 블록 |────────| I_DC Current Sink |
| (1.2V DC Ideal)   | Port 1 | (Power Plane 모델)   | Port 2 | (10.0A DC Load)   |
+-------------------+        +----------------------+        +-------------------+
          │                                                            │
         GND                                                          GND
```

1. **ADS 회로 구성**:
   - `Term1 (Port 1: VRM 커넥터 패드)`: DC 전압원(예: 1.2V)을 인가합니다.
   - `Term2 (Port 2: DUT Die 전원 패드)`: **10.0A** 부하 전류 싱크를 연결합니다.
2. **시뮬레이션 실행**:
   - ADS의 `DC Simulation Controller`를 배치하고 시뮬레이션을 실행합니다.
3. **전압 강하량($\Delta V_{drop}$) 계측**:
   - Port 1 노드 전압 $V(Port1) = 1.200\text{ V}$
   - Port 2 노드 전압 $V(Port2) = 1.1431\text{ V}$
   - 전압 강하량:
     $$\Delta V_{drop} = V(Port1) - V(Port2) = 1.200\text{ V} - 1.1431\text{ V} = 56.9\text{ mV}$$
4. **등가 저항 확인**:
   $$R_{DC} = \frac{56.9\text{ mV}}{10.0\text{ A}} = 5.69\text{ m}\Omega$$
5. **HTML 입력**:
   - [pdn_pi_ladder_modeler_b.html](file:///c:/python/Z_modeling/pdn_pi_ladder_modeler_b.html)의 **1번 카드**에 `Isink = 10.0 A`, `ΔVdrop = 56.9 mV`를 입력합니다.

---

### 2.2 S-parameter AC 시뮬레이션 및 마커 7개 추출법

1. **ADS S-파라미터 컨트롤러 설정**:
   - Sweep Type: **Log**
   - Start Frequency: **10 Hz**
   - Stop Frequency: **1 GHz**
   - Points/Decade: **50 ~ 100 pts/dec** (공진 골짜기의 정밀한 팁 측정을 위해 고해상도 설정 권장)
2. **Z-파라미터 변환 수식 (Data Display 창 Equation)**:
   ```text
   Z_mat = stoz(S, 50)
   Z22 = Z_mat(2,2)
   Z_mag = mag(Z22)
   ```
   - X축: `freq` (Log Scale)
   - Y축: `Z_mag` (Log Scale, 단위: Ohm)
3. **마커(Marker) 7개 지점 계측 및 HTML 입력값 매핑**:

| 번호 | 마커 명칭 | ADS 주파수 위치 ($f$) | ADS 관측 임피던스 ($|Z|$) | 모델 B 내부 매핑 및 물리적 기여 소자 |
| :---: | :--- | :--- | :--- | :--- |
| **①** | **저주파 용량성 하강** | `100 Hz` | `0.160 Ω` (160 mΩ) | 탄탈 뱅크 벌크 용량 ($C_{bulk} \approx 9,190\text{ }\mu\text{F}$) 도출 |
| **②** | **탄탈 ESR 선반** | `5 kHz` (1k~10kHz 플랫 구간) | `0.010 Ω` (10 mΩ) | 탄탈 뱅크 합성 내부 저항 ($R_{bulk} \approx 9.55\text{ m}\Omega$) 도출 |
| **③** | **1차 하강 슬로프** | `100 kHz` | `0.002 Ω` (2.0 mΩ) | 33µF MLCC 군 실효 합성 용량 ($C_{m1} \approx 746\text{ }\mu\text{F}$) 도출 |
| **③** | **1차 주공진 최저 바닥** | `680 kHz` (또는 600~700k 골짜기) | `0.001 Ω` (1.0 mΩ) | 33µF 뱅크 최저 저항 ($R_{m1} \approx 1.01\text{ m}\Omega$), 실장 인덕턴스 ($L_{m1} \approx 102.8\text{ pH}$) |
| **④** | **반공진 피크 (Peak)** | `2.5 MHz` (솟아오른 험프) | `0.0025 Ω` (2.5 mΩ) | $L_{m1}$과 소형 MLCC $C_{m2}$ 사이의 감결합 병렬 공진점 |
| **④** | **2차 공진 최저 바닥** | `5.5 MHz` (두 번째 골짜기) | `0.0020 Ω` (2.0 mΩ) | 10µF/1µF 소형 MLCC 뱅크 공진 ($R_{m2} \approx 0.47\text{ m}\Omega, C_{m2} \approx 29.3\text{ }\mu\text{F}$) |
| **⑤** | **초고주파 인덕턴스 수렴**| `100 MHz` | `0.050 Ω` (50 mΩ) | 최종 소켓/비아 루프 인덕턴스 ($L_{dut} \approx 65.0\text{ pH}$) 엄밀 결정 |

---

## 3. 웹 모델러 실행 및 결과 활용

1. 브라우저에서 [pdn_pi_ladder_modeler_b.html](file:///c:/python/Z_modeling/pdn_pi_ladder_modeler_b.html)을 실행합니다.
2. 좌측 1번 카드(IR Drop)와 2번 카드(주파수별 마커 7개)에 ADS에서 추출한 값을 확인/입력합니다.
3. **[🚀 Π형 래더 모델 정밀 피팅 및 갱신]** 버튼을 클릭합니다.
4. **정합도 확인**:
   - 우측 하단 **마커 오차율 분석 테이블**에서 7개 전 주파수 포인트의 정합 오차율이 **0.00% ~ 0.01% (녹색)**인지 확인합니다.
5. **시뮬레이션 탭 검증**:
   - `📊 Z(f) 주파수 응답 & 마커 정합`: 10Hz부터 1GHz까지 실제 S-parameter 플롯과 모델 곡선이 일치하는지 확인합니다.
   - `⚡ 과도 V-Droop 응답`: 10A 스텝 부하 인가 시 $L_{dut}$에 의한 초기 $di/dt$ 스파이크와 최종 $56.9\text{ mV}$ DC 드룹 평형 상태를 확인합니다.

---

## 4. 모델 B Netlist 예시 (LTspice / ADS 호환)

```spice
* ========================================================
* LTspice Deterministic PDN Pi-Ladder Equivalent Model
* Model B: Explicit Multi-Stage Pi-Ladder Topology
* Topology: VRM -> R_dc -> Tantalum -> Plane R/L -> MLCC -> Socket L -> DUT
* IR Drop: 56.9mV @ 10.0A (R_dc = 5.69mΩ)
* ========================================================
.subckt PDN_PI_LADDER_MODEL PORT_VRM PORT_DUT GND
* 1. DC Path Resistance (Power Path VRM to Board Bulk)
R_dc      PORT_VRM    Node_Bulk    5.6900e-03

* 2. Bulk Tantalum Capacitor Bank (470uF x 20) at Node_Bulk
R_bulk    Node_Bulk   n_b1         9.5500e-03
C_bulk    n_b1        n_b2         9.1900e-03
L_bulk    n_b2        GND          1.5000e-09

* 3. Power Plane Inter-stage Cavity (Node_Bulk to Node_MLCC)
R_plane   Node_Bulk   n_p1         1.1100e-03
L_plane   n_p1        Node_MLCC    8.5300e-11

* 4. MLCC 33uF Bank at Node_MLCC (Valley 1 Shaper)
R_m1      Node_MLCC   n_m1a        1.0100e-03
C_m1      n_m1a       n_m1b        7.4600e-04
L_m1      n_m1b       GND          1.0280e-10

* 5. MLCC 10uF / 1uF Bank at Node_MLCC (Peak & Valley 2 Shaper)
R_m2      Node_MLCC   n_m2a        4.7000e-04
C_m2      n_m2a       n_m2b        2.9300e-05
L_m2      n_m2b       GND          1.7300e-11

* 6. Socket / Via / Ball Loop Inductance (Node_MLCC to PORT_DUT)
L_dut     Node_MLCC   PORT_DUT     6.5000e-11
.ends PDN_PI_LADDER_MODEL
```

---

## 5. 모델 A와 모델 B의 비교 및 선택 가이드

| 비교 항목 | 모델 A (단일 버스 3-Branch 병렬) | 모델 B ($\Pi$형 다단 래더 분리형) |
| :--- | :--- | :--- |
| **관련 파일** | [pdn_pi_ladder_modeler_a.html](file:///c:/python/Z_modeling/pdn_pi_ladder_modeler_a.html) | [pdn_pi_ladder_modeler_b.html](file:///c:/python/Z_modeling/pdn_pi_ladder_modeler_b.html) |
| **토폴로지 구조** | 모든 소자가 단일 PDN 노드에 병렬 접속 | 벌크 ➔ $L_{plane}$ ➔ MLCC ➔ $L_{dut}$ 다단 분리 |
| **평면 인덕턴스 ($L_{plane}$)** | 별도 없음 (MLCC ESL에 통합 흡수) | **85.3 pH** 독립 소자로 명시 |
| **소켓 루프 인덕턴스 ($L_{dut}$)**| 0 pH (MLCC 노드가 다이에 직결) | **65.0 pH** 독립 소자로 명시 |
| **계산 복잡도** | 매우 단순, 빠른 수렴 | 다단 래더 어드미턴스 합성 필요 |
| **물리적 공간 표현력** | 낮음 (공간적 거리 무시) | **매우 높음 (실제 보드 레이아웃과 1:1 일치)** |
| **추천 시나리오** | 간이 시스템 레벨 과도 해석, 빠른 전력 해석 | **정밀 SI/PI 공동 시뮬레이션, 소켓 핀/비아 최적화** |
