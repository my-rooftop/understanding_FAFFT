# Improved Frobenius FFT for Code-Based Cryptography on Cortex-M4

- **저자**: Myeonghoon Lee, Jihoon Jang, Suhri Kim, Seokhie Hong (Korea University, Sungshin Women's University)
- **학회**: IEEE Internet of Things Journal, Vol. 12, No. 17, September 2025
- **키워드**: FAFFT, Cortex-M4, BIKE, HQC, polynomial multiplication, post-quantum cryptography

## 요약

ARM Cortex-M4 임베디드 플랫폼에서 **Frobenius Additive FFT (FAFFT)**의 butterfly 연산을 최적화하여 BIKE와 HQC의 다항식 곱셈 성능을 크게 개선한 연구. NIST가 임베디드 벤치마크 플랫폼으로 권장하는 Cortex-M4에서의 **최초 HQC 최적화 구현**.

## 핵심 문제

FAFFT의 butterfly 연산(Btfy)에서 수행되는 **유한체 곱셈**이 가장 큰 병목. Cortex-M4는 32-bit 아키텍처로 레지스터가 제한적(GP 14개, FP 32개)이어서 효율적 구현이 어렵다.

## 주요 기여

### 1. Butterfly 연산의 이진 행렬 곱셈 재해석
- FAFFT의 Btfy에서 곱셈의 한쪽 피연산자(multiplier)가 **고정값**(Cantor basis 원소의 조합)인 점에 착안.
- 유한체 곱셈을 **32x32 이진 행렬-벡터 곱셈(binary matrix multiplication)**으로 재구성.
- 이는 대칭키 암호의 **linear layer** 연산과 구조적으로 동일 -> 기존 최적화 기법 적용 가능.

### 2. XOR 연산 최소화
- 대칭키 암호 분야의 **s-XOR 최적화 도구** [Xiang et al.]를 적용하여 이진 행렬 곱셈의 XOR count 최소화.
- 각 multiplier(v17~v27)별로 기존 대비 **49~76개 s-XOR 절감**.
- 실제 clock cycle 기준으로 multiplier당 **225~289 cycle 절감**.

### 3. Cortex-M4 레지스터 스케줄링 전략
- GP 레지스터(14개)만으로는 32개 입력값의 XOR 시퀀스를 처리할 수 없음.
- **FP 레지스터(32개)를 저장소로 활용**: 32개 입력값을 FP 레지스터에 로드 -> 필요 시 GP로 이동(vmov) -> XOR 수행 -> 다시 FP로 반환.
- vmov와 XOR 모두 1 cycle이므로 데이터 이동 오버헤드 최소화.

### 4. Bitslice 구현
- Encode 과정에서 F2 계수를 F_{2^32}로 변환 시 bitslice 형태로 처리.
- 32개 계수를 하나의 32-bit 워드에 패킹하여 병렬 처리.

## 타겟 플랫폼
- **STM32L4R5ZI** (NUCLEO 보드): ARM Cortex-M4, 2MB Flash, 640KB SRAM
- 컴파일러: arm-none-eabi-gcc 10.3.0
- 클럭: 20MHz (pqm4 프레임워크 설정)

## 성능 결과

### BIKE 다항식 곱셈 (cycle counts)
| | 기존 [Chen&Chou] | 본 연구 | 개선율 |
|---|---|---|---|
| BIKE-1 | 1,598,737 | 1,475,953 | ~8% |
| BIKE-3 | 3,509,004 | 3,238,668 | ~8% |

### BIKE 전체 연산
| | key gen. | encap. | decap. |
|---|---|---|---|
| BIKE-1 기존 | 26,921,361 | 3,397,581 | 58,151,362 |
| BIKE-1 본 연구 | 24,915,790 | 3,274,791 | 56,840,529 |
| 개선율 | ~6% | ~3% | ~2% |

### HQC 다항식 곱셈 (vs PQClean 32-bit 포팅)
| | PQClean [14] | 본 연구 | 개선율 |
|---|---|---|---|
| HQC-128 | 12,846,453 | 2,888,842 | ~78% |
| HQC-192 | 39,412,568 | 6,361,943 | ~84% |
| HQC-256 | 75,879,677 | 6,372,501 | ~92% |

### HQC 전체 연산 (vs PQClean)
| | key gen. | encap. | decap. | 개선율 |
|---|---|---|---|---|
| HQC-128 본 연구 | 4,201,940 | 8,657,454 | 14,075,463 | ~70% |
| HQC-192 본 연구 | 9,771,360 | 19,845,917 | 31,170,491 | ~77% |
| HQC-256 본 연구 | 12,757,862 | 25,793,534 | 41,868,167 | ~84% |

## FAFFT 알고리즘 (F_{2^32} 기준, m=32)

1. **BasisCvt**: monomial basis -> novel polynomial basis 변환
2. **Encode**: Frobenius partition Sigma = v_{k+16} + W_k에서의 truncated evaluation. 계수를 F2에서 F_{2^32}로 bitslice 변환.
3. **Btfy**: butterfly 연산. 재귀적으로 다항식을 분할하며 evaluation 수행. 핵심 곱셈을 이진 행렬 곱셈으로 최적화.

## 핵심 인사이트
- FAFFT butterfly의 고정 multiplier 특성을 이용해 **유한체 곱셈 -> 이진 행렬 곱셈** 변환이 핵심 아이디어.
- 대칭키 암호의 linear layer 최적화 기법이 공개키 암호(code-based)에도 적용 가능함을 보임.
- XOR 시퀀스 최적화 자체는 아키텍처 독립적이나, 레지스터 스케줄링은 Cortex-M4 특화.
- HQC-256에서 polymul이 HQC-192와 비슷한 cycle인 이유: FAFFT는 2^17 고정 차수로 곱셈하므로 HQC-192(35851)와 HQC-256(57637) 모두 같은 FAFFT 크기 사용.
