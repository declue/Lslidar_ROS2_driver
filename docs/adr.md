# ADR-0001: Lslidar ROS2 Driver 멀티스레드 데이터 파이프라인 개선

- **상태:** 제안 (Proposed)
- **대상 브랜치:** M10P/N10P
- **작성일:** 2026-04-14
- **적용 범위:** `lslidar_driver` 패키지의 수신/처리/발행 파이프라인
- **근거 문서:** `docs/architecture.md`, `docs/thread-analysis.md`, `docs/analysis.md`, `docs/code-metrics.md`

---

## 1. 컨텍스트와 문제 정의

현재 드라이버는 생산자-소비자 패턴의 2-스레드 구조로 동작한다 (`thread-analysis.md` §스레드 아키텍처).

- **메인 스레드 (`polling()`)**: UDP/시리얼 수신 → 헤더 검증 → CRC → 파싱 → 좌표 변환 → 버퍼 저장 → 알림을 모두 단일 함수에서 수행
- **발행 스레드 (`pubScanThread()`)**: 조건 변수 대기 → 버퍼 복사 → LaserScan/PointCloud2 변환 → 토픽 발행
- **동기화**: 단일 전역 뮤텍스 `mutex_` + 조건 변수 `pubscan_cond_`

### 식별된 구조적 한계

| # | 한계 | 근거 |
|---|---|---|
| L1 | `polling()` 함수가 I/O·검증·파싱·변환·동기화까지 5개 책임을 단일 흐름에서 처리 | `thread-analysis.md` §polling() 함수 |
| L2 | 패킷 수신 시마다 `new unsigned char[500]` 동적 할당, 스캔 완료마다 `scan_points_bak_.resize()`/`assign()` 반복 | `thread-analysis.md` §data_processing() |
| L3 | 단일 뮤텍스가 `scan_points_` 전체 벡터를 보호 → 복사 구간 동안 수신 스레드도 대기 | `thread-analysis.md` §뮤텍스 |
| L4 | `wait_for_wake` 플래그 기반 조건 변수 대기 — spurious wakeup에 대한 predicate 기반 대기가 아님 | `thread-analysis.md` §스피너(Spurious Wakeup) |
| L5 | 발행 스레드에서 모델이 `N10_P`/`M10_DOUBLE`이 아니면 조건 변수로 깨어나도 아무 일도 하지 않고 재대기 (헛깨움 비용) | `thread-analysis.md` §pubScanThread() |
| L6 | `lslidar_driver.cc` 1,383줄·주석 2.5%·테스트 0% — 변경 리스크가 큼 | `code-metrics.md` §파일별 분석 |

### 운용 기준선 (baseline)

`analysis.md` §성능 특성 및 `thread-analysis.md` §성능 특성에 기록된 현재 수치:

| 지표 | 현재값 |
|---|---|
| 처리량 | 약 400K points/s |
| 포인트/스캔 | 약 20,000 |
| 스캔률 | 10–20 Hz |
| 지연 시간 | < 50 ms |
| 락 경합 시간 | < 1 ms |
| 발행 스레드 대기 | 0–100 ms |
| 패킷 손실률 | < 1 % |
| CPU 사용량 | < 20 % |
| 메모리 사용량 | < 100 MB |

---

## 2. 품질 속성과 품질 시나리오

### 2.1 대상 품질 속성 (Quality Attributes)

본 ADR에서 결정의 판단 기준이 되는 **비기능적 요구사항(ISO/IEC 25010 기반)**. 기능은 이미 구현돼 있으므로 구조 선택은 전적으로 아래 속성들로 평가한다.

| 품질 속성 | 본 프로젝트에서의 의미 | 관련 근거 문서 |
|---|---|---|
| **Performance** (성능) | 처리량·지연·락 경합 시간. 라이다는 실시간 센서라 1차 속성 | `analysis.md` §성능 특성, `thread-analysis.md` §성능 특성 |
| **Reliability** (신뢰성) | 패킷/스캔 손실 최소화, 데이터 정책의 일관성 | `analysis.md` §통신 상태 (손실률 < 1%) |
| **Modifiability** (변경 용이성) | 1,383줄 단일 파일·주석 2.5%·테스트 0%인 상황에서 후속 변경 가능성 확보 | `code-metrics.md` §파일별 분석, §유지보수성 지표 |
| **Testability** (시험 용이성) | 현재 테스트 커버리지 0% — Phase 2/3 진입 전제 조건 | `code-metrics.md` §테스트 커버리지, `analysis.md` §테스트 전략 |
| **Observability** (관찰 가능성) | 변경 효과를 런타임에 계측 가능한가 (`diagnostic_updater` 확장) | `architecture.md` §진단 시스템, `analysis.md` §진단 시스템 상세 |

> 본 문서 범위에서 제외: **Security**(내부망 폐쇄 운용 가정 — `analysis.md` §보안 고려사항), **Usability**(엔드유저 UI 없음), **Interoperability**(ROS2 표준 메시지 준수로 이미 확보).

### 2.2 속성 → 시나리오 매핑

| 시나리오 | 주 품질 속성 | 부 품질 속성 |
|---|---|---|
| **QS-A** 메인 스레드 처리량 | Performance (throughput) | Reliability |
| **QS-B** 발행 처리량·지연 | Performance (latency) | Reliability |
| **QS-C** 락 대기 시간 | Performance (contention) | Reliability |
| **QS-D** 생산-소비 불일치 | Reliability (데이터 정책) | Observability |
| **QS-E** 코드 변경 용이성 | Modifiability | Testability |

### 2.3 품질 시나리오 (Quality Attribute Scenarios)

SMART 형식(자극원·자극·응답·응답측정)으로 기술한다. 모든 목표치는 baseline 대비 **유지 또는 개선**.

### QS-A: 메인(수신·처리) 스레드 처리량

**주 품질 속성:** Performance (throughput) / **부 품질 속성:** Reliability

| 항목 | 내용 |
|---|---|
| **자극원** | Leishen M10/N10 계열 라이다 |
| **자극** | 10–20 Hz 스캔률로 MSOP 패킷 연속 도착 |
| **환경** | 정상 운전 (CPU < 20%, 손실률 < 1%) |
| **응답** | 패킷 수신 → 버퍼 저장까지 파이프라인 유지 |
| **응답 측정** | **처리량 ≥ 400K points/s, 포인트 드롭 < 1%** (baseline 유지) |

### QS-B: 발행 스레드 처리량

**주 품질 속성:** Performance (latency) / **부 품질 속성:** Reliability

| 항목 | 내용 |
|---|---|
| **자극원** | 메인 스레드의 `notify_one()` |
| **자극** | 스캔 완료 이벤트 (10–20 Hz) |
| **환경** | LaserScan + PointCloud2 동시 발행 활성 |
| **응답** | 토픽 발행 완료 |
| **응답 측정** | **발행률 ≥ 스캔률(10–20 Hz) × 100%, 수신→발행 end-to-end 지연 < 50 ms** |

### QS-C: 공유 메모리 락 대기 시간

**주 품질 속성:** Performance (contention) / **부 품질 속성:** Reliability

| 항목 | 내용 |
|---|---|
| **자극원** | 발행 스레드의 `getScan()` 호출 |
| **자극** | 수신 스레드가 `mutex_` 보유 중 |
| **환경** | 스캔 완료 시점의 버퍼 복사 구간 |
| **응답** | `lock()` 획득 |
| **응답 측정** | **락 획득 대기 시간 p99 ≤ 1 ms** (baseline 유지), 대기 중 패킷 드롭 없음 |

### QS-D: 생산-소비 속도 불일치 대응

**주 품질 속성:** Reliability (데이터 정책) / **부 품질 속성:** Observability

| 항목 | 내용 |
|---|---|
| **자극원** | 시스템 부하 증가 또는 발행 스레드 일시 지연 |
| **자극** | 발행 스레드 처리 속도 < 메인 스레드 공급 속도 |
| **환경** | 단일 백업 버퍼 `scan_points_bak_`만 존재 (현재 구조) |
| **응답** | 최신 스캔만 보존, 오래된 데이터는 버림 |
| **응답 측정** | **최신 스캔 보존율 100%, 스캔 건너뜀 발생 시 로그 기록**, 버퍼 메모리 O(1) 유지 |

### QS-E: 코드 변경 용이성

**주 품질 속성:** Modifiability / **부 품질 속성:** Testability

| 항목 | 내용 |
|---|---|
| **자극원** | 개발자 (신규 라이다 모델 지원·버그 수정·리팩토링) |
| **자극** | `lslidar_driver.cc` 내부 로직 변경 요청 |
| **환경** | 현재 baseline — 단일 파일 1,383줄, 주석 2.5%, 테스트 0% (`code-metrics.md`) |
| **응답** | 변경이 국소화되고 회귀가 자동 검증됨 |
| **응답 측정** | **단일 책임 함수당 ≤ 100 줄**, `polling()` 분해 후 **함수당 복잡도 "중간" 이하**, 핵심 경로 **통합 테스트 1건 이상 존재** |

> **주의 — 원본 초안 수정 내역**
> - `writeㅅ속도` 오타 → `write 속도` 수정
> - 원문 "공유 메모리 사이즈 증가" 표현 수정: 현재 구현은 단일 백업 버퍼라 크기 증가가 발생하지 않으며, 실제 현상은 "덮어쓰기로 인한 스캔 유실"이다. QS-D는 "유실 허용 + 최신성 우선" 정책을 명시한다.

---

## 3. 개선 후보 구조 설계

각 품질 시나리오에 대응되는 후보를 제시한다. 범위는 드라이버 내부 구조로 한정한다(라이다 측 프로토콜은 변경 불가 — 초안의 "통신 프로토콜 변경" 항목은 제외).

### A1. 메인 스레드 책임 분리 + 메모리 풀

**대응 시나리오:** QS-A (주), QS-E (부 — 함수 분해로 Modifiability 확보)

1. **3-단계 파이프라인 분리**
   - 스레드 1: 순수 I/O (UDP/시리얼 수신)
   - 스레드 2: 검증(헤더·CRC) + 파싱 + 좌표 변환
   - 스레드 3: 발행 (기존 `pubScanThread` 유지)
2. **패킷 버퍼 풀링**: 고정 크기(`N × 500B`) 풀을 선할당하여 `new/delete` 제거 (L2 해소)
3. **단일 책임 함수화**: `polling()` → `receivePacket()` / `validatePacket()` / `parsePacket()` / `transformCoords()` 분리 (L1·L6 해소)

### B1. 발행 스레드 분화 + 배치

**대응 시나리오:** QS-B

1. **토픽별 발행 스레드 분리**: LaserScan 전용 / PointCloud2 전용 (현재는 동일 스레드에서 순차 발행)
2. **모델별 dead branch 제거**: 지원 모델이 아닌 경우 발행 스레드를 생성하지 않음 (L5 해소)
3. **배치 발행 옵션**: 실시간성이 불필요한 운용 모드에서 N개 스캔 묶어 발행 (기본 OFF, 파라미터화)

> 초안의 "통신 프로토콜 변경" 제안은 라이다 펌웨어 영역이므로 드라이버 ADR 범위에서 제외한다.

### C1. 락 그레인 세분화 + 경량 동기화

**대응 시나리오:** QS-C (주), QS-B (부 — 지연 감소)

1. **버퍼 포인터 스왑**: 단일 뮤텍스 대신 `std::atomic<ScanBuffer*>`로 활성/백업 포인터만 교환 (복사 없이 O(1))
2. **SPSC(Single-Producer Single-Consumer) 링 버퍼**: 메인→발행 채널을 lock-free SPSC queue로 대체
3. **Predicate 기반 `condition_variable::wait`**: `wait_for_wake` 플래그를 제거하고 표준 `wait(lock, pred)` 사용 (L4 해소)

> 초안의 "mutex → queue 전환"은 "lock-free SPSC 링 버퍼"로 구체화.

### D1. 드롭 정책 + 중복 제거

**대응 시나리오:** QS-D (주), QS-E (부 — `dropped_scans` 계측으로 Observability 확보)

1. **최신-우선(Latest-Wins) 정책**: 더블 버퍼 포인터 스왑 시 이전 백업이 미소비 상태면 덮어쓰고 `dropped_scans` 카운터 증가
2. **오래된 데이터 폐기**: 발행 스레드가 깨어났을 때 타임스탬프 기준 임계(기본 2 × 스캔 주기) 초과면 폐기
3. **라이다 특성상 중복 포인트 제거**: 동일 방위각 연속 수신 시 최신값만 유지 (`data_processing()` 내 idx 덮어쓰기 로직 보강)

---

## 4. 후보 구조 평가

### 평가 축과 가중치

각 축을 2.1절의 **품질 속성**과 명시적으로 연결한다.

| 축 | 품질 속성 대응 | 가중치 | 정의 |
|---|---|---:|---|
| 성능 개선 효과 | **Performance** | 0.30 | 해당 시나리오 지표 개선 폭 |
| 신뢰성 효과 | **Reliability** | 0.15 | 손실·드롭·stale 데이터 제어 수준 |
| 회귀 위험(역, 반전) | **Modifiability** (현재) | 0.20 | `lslidar_driver.cc` 중심부 변경 여부, 테스트 부재 반영 (`code-metrics.md` 테스트 0%) |
| 구현 복잡도(역, 반전) | Cost/Effort | 0.15 | 변경 LOC·신규 개념 수 (낮을수록 고득점) |
| 관찰 가능성 | **Observability** | 0.10 | 변경 효과를 `diagnostic_updater`로 계측 가능한가 |
| 장기 변경 용이성 | **Modifiability/Testability** | 0.10 | 함수 분해·테스트 가능한 경계 도입 여부 |

점수: 1(낮음) – 5(높음). 가중합 만점 5.0.

### 평가 매트릭스

점수: 1(낮음) – 5(높음). "회귀위험"·"복잡도"는 값이 클수록 고득점(즉, 위험/복잡도가 낮을수록 5점). 가중합 만점 5.0.

| 후보 | Perf.(0.30) | Reli.(0.15) | Modif.회귀↓(0.20) | Cost↓(0.15) | Obs.(0.10) | Modif.장기(0.10) | **가중합** |
|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| **A1** 스레드 분리 + 풀 | 4 | 4 | 2 | 2 | 4 | 5 | **3.30** |
| **B1** 발행 분화 + 배치 | 3 | 3 | 4 | 3 | 3 | 3 | **3.20** |
| **C1** 락 세분화 + SPSC | 5 | 4 | 3 | 3 | 5 | 3 | **3.80** |
| **D1** 드롭 정책 + 중복제거 | 3 | 5 | 4 | 4 | 5 | 3 | **3.85** |

### 주요 관찰 (품질 속성 관점)

- **D1·C1 공동 최상위**: D1은 Reliability·Observability에서 최고점 — baseline에 이미 존재하는 손실률 < 1% 지표(`analysis.md`)에 `dropped_scans` 계측을 더해 신뢰성을 **증명 가능한 속성**으로 격상.
- **C1은 Performance 축에서 단일 최상위**: 포인터 스왑은 락 경합 시간을 상수로 바운드 — `thread-analysis.md`의 "락 경합 < 1 ms" 기준을 p99로 격상 가능.
- **A1은 Modifiability 장기 이득 최상위(5점)**: 함수 분해는 QS-E 직접 대응. 다만 테스트 0% 환경에서 회귀 위험이 2점으로 최하 — **Testability 선행 없이는 실행 불가**.
- **B1은 전 축 평균적**: 단일 killer 효과 없음. 단, B1-2(dead branch 제거)는 Phase 1에 분리 수용 가능.

---

## 5. 결정

다음 **단계적 도입**을 채택한다.

### Phase 1 (즉시, 저위험) — 채택

**목표 품질 속성:** Reliability, Observability (Performance는 유지)

- **C1-3**: `condition_variable::wait(lock, predicate)`로 전환 (L4 해소) — Reliability
- **B1-2**: 미지원 모델에서 발행 스레드 미생성 (L5 해소) — Performance(불필요 깨움 제거)
- **D1-1**: `dropped_scans` 카운터 추가 + `diagnostic_updater` 연결 — Observability
- **D1-2**: 스캔 타임스탬프 기반 stale 데이터 폐기 — Reliability
- **(신규) E1-0**: 핵심 경로 통합 테스트 1건 시드(PCAP 재생 기반) — Testability 진입점 (`analysis.md` §테스트 전략 참조)

**근거:** 구현 복잡도 낮음, `lslidar_driver.cc` 중심 로직 변경 최소, Observability 즉시 확보. `code-metrics.md`의 테스트 0% 상황에서 E1-0을 Phase 1에 포함시켜 Phase 2/3 진입 조건(Testability)을 선제 구축.

### Phase 2 (테스트 기반 확보 후) — 조건부 채택

**목표 품질 속성:** Performance (contention, latency)

- **C1-1**: 더블 버퍼 포인터 스왑 (`std::atomic<ScanBuffer*>`)
- **C1-2**: SPSC 링 버퍼 도입

**선행 조건 (Testability):** Phase 1의 E1-0을 기반으로 통합 테스트 계층(`analysis.md` §테스트 전략)을 확장하여 수신→발행 end-to-end 회귀 테스트 확보.

### Phase 3 (장기) — 조건부 채택

**목표 품질 속성:** Modifiability, Performance (throughput 여유)

- **A1-1/2/3**: 3-스레드 파이프라인 분리 + 패킷 풀 + `polling()` 함수 분해

**선행 조건:** Phase 2 완료 및 성능 프로파일(`analysis.md` §성능 특성)로 A1 필요성 재확인. 현재 baseline이 이미 지연 < 50 ms, CPU < 20 %이므로 Performance 근거 없이 Modifiability만으로는 선투자하지 않는다. **QS-E 재평가 후 진입**.

### 기각

- **B1-1 (토픽별 발행 스레드 분리)**: 현재 발행 비용(`thread-analysis.md` §동기화 오버헤드: 메시지 변환 10ms + 발행 5ms)이 스캔 주기(50–100 ms) 대비 충분한 여유 — 분리 이득 작고 스레드 수만 증가.
- **B1-3 (배치 발행)**: LiDAR는 실시간 센서 — 일괄 처리는 QS-B의 end-to-end 지연 목표와 충돌.
- **초안 B1의 "통신 프로토콜 변경"**: 드라이버 범위 밖.

---

## 6. 결과 (Consequences)

### 긍정적

- **Reliability↑**: Spurious wakeup 표준 대응으로 조건 변수 사용이 견고해짐, stale 데이터 폐기로 데이터 정책 일관성 확보
- **Performance↑ (미세)**: 미지원 모델 dead branch 제거로 불필요 컨텍스트 스위치 제거
- **Observability↑**: `dropped_scans` / stale drop 계측으로 QS-D 충족 여부 런타임 가시화
- **Testability↑**: Phase 1의 E1-0 통합 테스트 시드로 Phase 2/3 진입 경로 확보

### 부정적 / 리스크

- **Interoperability 영향**: `diagnostic_updater` 항목 추가로 진단 토픽 스키마 소폭 변경 — 소비자 측 호환성 확인 필요
- **Performance 체감 변화 미미**: Phase 1에서는 성능 수치 자체는 크게 변하지 않음 (주 목적은 Reliability·Observability)
- **Modifiability 장기 부채**: Phase 3(A1)까지 진입하지 않으면 `lslidar_driver.cc` 1,383줄·주석 2.5% 상태가 잔존 — QS-E는 부분 충족에 그침

### 검증 방법

| 시나리오 | 품질 속성 | 검증 지표 | 수집 방법 |
|---|---|---|---|
| QS-A | Performance | 포인트 드롭 < 1% | `packet_loss` 진단 항목(`analysis.md` §통신 상태) |
| QS-B | Performance | 토픽 발행률 = 스캔률 | `diag_min_freq`/`diag_max_freq`(`architecture.md` §진단 시스템) |
| QS-C | Performance | 락 획득 대기 p99 ≤ 1 ms | `thread-analysis.md` §락 경합 시간 측정 코드 패턴 적용 |
| QS-D | Reliability | `dropped_scans` 증가율 관찰 | 신규 진단 항목 |
| QS-E | Modifiability/Testability | 통합 테스트 건수 ≥ 1, `polling()` 분해 후 함수당 LOC ≤ 100 | Phase 1 E1-0 테스트 스위트, `cloc` 재측정(`code-metrics.md` 형식) |

---

## 7. 참고

- `docs/architecture.md` — 시스템 아키텍처, 진단 시스템
- `docs/thread-analysis.md` — 스레드 동작, 동기화, 성능 특성, 잠재적 문제
- `docs/analysis.md` — 성능 특성, 테스트 전략, 향후 개선 방향
- `docs/code-metrics.md` — 코드 규모, 복잡도, 테스트 커버리지 현황
