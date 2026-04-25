# 3-Sphere Omniwheel Mobility Platform

> STM32 기반 전방향 이동 모빌리티 | 조이스틱·음성인식 듀얼 HMI  
> 2025 한국 Maker Faire 과학기술정보통신부 장관상 (최우수상)

---

## 프로젝트 개요

3개의 구형 바퀴(Sphere Wheel)를 이용해 전방향 이동이 가능한 모빌리티를 설계·제작했습니다.  
조이스틱 수동제어와 Faster-Whisper STT 기반 음성인식 자동제어를 동시 지원하는  
듀얼 HMI 구조로 구현했습니다.

개발 과정에서 발생한 RF 통신 불안정과 저속 진동 문제를  
오실로스코프·데이터 로그 기반으로 진단해 PCB 재설계와 제어 파라미터 조정으로 해결했습니다.

---

## 완성 결과물
<img width="1960" height="1016" alt="Image" src="https://github.com/user-attachments/assets/7a624965-8e17-4323-af49-88438761a222" />

---

## 시스템 아키텍처
<img width="933" height="194" alt="Image" src="https://github.com/user-attachments/assets/bbef907b-a5ea-48a1-9647-8179b3cfcac3" />

---

## 저장소 구조

이 프로젝트의 펌웨어는 Transmit / Receive 두 파트로 분리되어 있습니다.

| 폴더 | 설명 |
|------|------|
| `Transmit/` | 조이스틱 ADC 수집 + RF 송신 STM32 펌웨어 |
| `Receive/` | RF 수신 + CAN Bus로 VESC 3개 제어 STM32 펌웨어 |

---

## Branch 구조

### Transmit

| Branch | 설명 |
|--------|------|
| `master` | 폴링 기반 ADC + HAL_Delay(20ms) RF 송신 |
| `feature/interrupt-conversion` | TIM2(50Hz) + ADC DMA 3채널 동시 수집 + WFI 저전력 구조로 전환 |
| `State_Machine` | UART3로 RPI5 음성인식 명령 수신<br>ST_IDLE / ST_JOYSTICK / ST_VOICE 3상태 머신 구현<br>VOICE_MAP으로 방향 명령 → ADC값 매핑 |

### Receive

| Branch | 설명 |
|--------|------|
| `master` | 폴링 기반 RF 수신 + TIM1 PWM으로 VESC 3개 PPM 제어 |
| `interrupt-conversion` | nRF24 IRQ 인터럽트 기반 수신 구조로 전환 |
| `kiwiDrive_Modify` | PPM → CAN Bus 전환 (ERPM 기반 속도 제어)<br>TIM3 Watchdog (100ms 무신호 시 모터 자동 정지)<br>E-Stop 래치 기능 추가<br>저속 진동 원인을 역기구학으로 오판, 코드 수정 시도 흔적<br>(실제 원인: 모터 제조 공차 + PID 설정 한계로 해결) 

---

## 문제 해결 기록

### Episode 1 — RF 통신 끊김을 전원 노이즈로 진단해 통신 안정성 99% 확보

| | |
|---|---|
| **문제** | 브레드보드 배선 복잡화 → RF 통신 불규칙 끊김, ADC 값 흔들림 |
| **진단** | 오실로스코프 전원 라인 계측 → 모터 구동 시 전압 강하·노이즈 확인 |
| **해결** | LDO(AP2112K), 디커플링 커패시터, RC 필터 적용 PCB 쉴드 직접 설계·제작 |
| **결과** | 통신 안정성 **99%** 확보, ADC 지터 약 **90%** 감소 |

---

### Episode 2 — 모터 공차를 데이터로 진단해 저속 제어 안정화

| | |
|---|---|
| **문제** | 저속 주행 시 진동·모터 멈춤 반복, 역기구학·코드 수정으로 미해결 |
| **진단** | VESC Tool로 3개 모터 ERPM 동시 수집 → 모터 간 응답 편차, 저속 구간 전류 스파이크 확인 |
| **해결** | Minimum ERPM 900 → 50, PID Kp 0.00400 → 0.00075 재설정 |
| **결과** | 저속 구간 전류 스파이크 **10A+ → 0.7A 이내** 감소, 30회 반복 검증 |

> `kiwiDrive_Modify` 브랜치: 역기구학 오류로 처음 오판했던 코드 수정 시도 흔적이 남아있습니다.

---

### Episode 3 — PPM 1:1 구조를 CAN Bus 데이지체인으로 전환

| | |
|---|---|
| **문제** | PPM 방식 배선 9가닥, 단방향 구조, 다중 노드 제어·상태 모니터링 불가 |
| **해결** | STM32 + VESC 3개 데이지체인 CAN Bus 구조 전환, 종단저항 배치, 하네스 직접 제작 |
| **결과** | 배선 **9가닥 → 3가닥**, 양방향 통신, 9시간 37팀 현장 운용 통신 단절 없음 |

---

## 사용 기술

| 분류 | 내용 |
|------|------|
| MCU | STM32F103 x2 (Transmit / Receive) |
| 통신 | nRF24L01 RF(SPI), CAN Bus, UART |
| 타이머 | TIM1(PWM 20ms), TIM2(ADC 트리거 50Hz), TIM3(Watchdog 50ms) |
| ADC | DMA 3채널 동시 수집 (X·Y·Z 조이스틱) |
| 제어 | Kiwi Drive 역기구학, ERPM 기반 속도 제어 |
| 안전 | E-Stop 래치, Watchdog 타임아웃 자동 정지 |
| 음성인식 | Faster-Whisper STT → UART3 → State Machine |
| PCB | KiCad·EasyEDA 직접 설계·발주 |
| 3D 설계 | Fusion360 (섀시·감속기·구형 바퀴) |

---

## Hardware

| 부품 | 사양 |
|------|------|
| MCU | STM32F103 x2 (Transmit / Receive) |
| BLDC Motor | BDUAV 5065 140KV x3 |
| ESC | Makerbase VESC MINI 6.7 PRO x3 |
| RF 모듈 | nRF24L01 |
| 배터리 | 3S LiPo 5000mAh |
| 상위 처리 | Raspberry Pi 5 |
| PCB | 자체 설계 쉴드 (KiCad, EasyEDA) |
| 감속기 | 풀리(HTD3M): 20T, 56T, 106T / 벨트(HTD3M): 279mm, 589mm |
| 섀시, 휠 | 자체 설계, 3D 출력 (PLA+) |

---

## 음성인식 제어

Raspberry Pi 5에서 Faster-Whisper STT로 음성 명령을 텍스트로 변환,  
UART3으로 STM32에 전달하는 State Machine 구조로 구현했습니다.

| 명령 | 동작 |
|------|------|
| 전진 | Y축 최대값 |
| 후진 | Y축 최소값 |
| 좌이동 | X축 최소값 |
| 우이동 | X축 최대값 |
| 정지 | 중립값 |
---

## 수상

- **2025 한국 Maker Faire 과학기술정보통신부 장관상 (최우수상)**
- 9시간 37팀 현장 운용 검증

---

## 관련 저장소

| 저장소 | 설명 |
|--------|------|
| [3sphere-wheel-transmit](https://github.com/xodrb/3sphere-wheel-transmit) | Transmit 펌웨어 (폴링→인터럽트→State Machine) |
| [3sphere-wheel-receive](https://github.com/xodrb/3sphere-wheel-receive) | Receive 펌웨어 (폴링→인터럽트, 모터 제어) |
