# Accelerating HQC with Additive FFT

- **저자**: Ming-Shing Chen, Chun-Ming Chiu, Chun-Tao Peng, Bo-Yin Yang (Academia Sinica, Taiwan)
- **학회**: TCHES 2026
- **키워드**: HQC, additive FFT, F2 polynomial multiplication, extended CRT

## 요약

NIST PQC 표준으로 선정된 HQC (Hamming Quasi-Cyclic) KEM의 핵심 연산인 **F2[x] 다항식 곱셈**을 additive FFT를 활용하여 가속하는 연구.

## 핵심 문제

HQC의 다항식 차수(n = 17669, 35851, 57637)가 **2의 거듭제곱이 아니므로** 표준 FFT 기반 다항식 곱셈을 직접 적용하기 어렵다.

## 주요 기여

### 1. 두 가지 FFT 기반 F2 다항식 곱셈 방법
- **Kronecker Segmentation (KS)**: 다항식을 m-bit 블록으로 분할 -> F_{2^m} 위의 다항식으로 lift -> addFFT로 곱셈. F_{256^2} 확장체 사용으로 8-bit 명령어(PMULL.P8) 활용 가능.
- **Frobenius Additive FFT (FAFFT)**: Frobenius map (x -> x^2)을 이용해 evaluation domain을 m/2배 축소. KS 대비 메모리/연산 약 절반. F_{2^64} 위에서 동작.

### 2. Additive FFT + CRT 결합 (핵심 신규 기법)
- 2의 거듭제곱이 아닌 차수 처리를 위해 **Chinese Remainder Theorem**을 도입.
- 다항식 링을 `F2[x]/s_i x F2[x]/x^c`로 분해하여, 큰 2의 거듭제곱 부분은 addFFT로, 나머지 작은 부분은 Karatsuba로 곱셈.
- HQC-1 예시: n=17669 -> 2^15 + 3072로 분할, s_12 기준 CRT 분해.

### 3. HQC 전체 최적화
- **Cached Key Transform**: 키 생성 시 h, s, y의 FFT 변환값을 미리 계산/저장 -> Encap/Decap에서 재사용.
- **Common Input Transform**: Encrypt에서 r2의 FFT 변환을 h*r2, s*r2 두 곱셈에 공유 -> 입력 변환 1회 절약 (x86 AVX2 기준 연산 시간 ~1/8, ARM NEON 기준 ~1/6 절감).
- **RM-RS 코덱 SIMD 최적화**: expand_and_sum 벡터화, F256 곱셈 최적화 (GFNI의 GF2P8MULB 활용).

### 4. 사이드 채널 저항성
- constant-time 구현 (메모리 접근/제어 흐름이 비밀값에 독립).
- 파워 사이드 채널 공격 대응 (SIMD 벡터화된 bit extraction).

## 타겟 플랫폼 및 명령어
| 플랫폼 | polymul 명령어 | FFT 방법 |
|--------|---------------|----------|
| x86 AVX2 | PCLMULQDQ (64x64->128) | FAFFT (F_{2^64}) |
| x86 AVX2+GFNI | GF2P8MULB (F256 native) | FAFFT (F_{256^2}) |
| Apple M1 (ARM NEON) | PMULL.P64 (64x64->128) | FAFFT (F_{2^64}) |
| Raspberry Pi4 (Cortex-A72) | PMULL.P8 (8x8->16) | KS + CRT (F_{256^2}) |

## 성능 결과 (다항식 곱셈, cycle counts)

| 플랫폼 | 알고리즘 | HQC-1 | HQC-3 | HQC-5 |
|--------|---------|-------|-------|-------|
| x86 AVX2 | Toom-Karatsuba (기존) | 27,662 | 80,881 | 195,234 |
| x86 AVX2 | FAFFT (F_{2^64}) | 57,612 | 139,629 | 202,792 |
| x86 GFNI | FAFFT (F_{256^2}) | 34,042 | 87,652 | 102,997 |
| Apple M1 | FAFFT | 39,032 | 92,186 | 135,700 |
| Pi4 | KS+CRT | 329,319 | 788,052 | 1,317,323 |

- GFNI 사용 시 PCLMULQDQ 대비 약 **2배 성능 향상** (HQC-5 기준).
- Pi4(임베디드)에서는 FFT 기반이 Toom-Karatsuba 대비 우위.
- x86 AVX2에서는 전용 64-bit polymul 명령어의 이점으로 Toom-Karatsuba와 근접한 성능.

## 핵심 인사이트
- FFT의 진정한 이점은 단순 polymul 속도가 아니라, **변환값 캐싱/재사용**을 통한 HQC 전체 연산 최적화에 있음.
- 알고리즘-ISA 공동 설계 (GFNI + F_{256^2} 체 선택)가 극적인 성능 향상을 가져옴.
- CRT 분해는 2의 거듭제곱이 아닌 차수를 FFT로 처리하는 범용적 기법.
