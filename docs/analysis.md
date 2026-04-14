# Lslidar ROS2 Driver 상세 분석

## 소스 코드 구조 분석

### 주요 소스 파일

```mermaid
graph TB
    subgraph "lslidar_driver/src/"
        NODE[lslidar_driver_node.cc<br/>메인 노드 진입점]
        DRIVER[lslidar_driver.cc<br/>드라이버 핵심 로직]
        INPUT[input.cc<br/>데이터 입력 처리]
        OSR[lsiosr.cpp<br/>시리얼 통신]
    end

    subgraph "lslidar_driver/include/lslidar_driver/"
        H_DRIVER[lslidar_driver.h<br/>드라이버 헤더]
        H_INPUT[input.h<br/>입력 클래스 헤더]
        H_OSR[lsiosr.h<br/>시리얼 클래스 헤더]
    end

    NODE --> H_DRIVER
    DRIVER --> H_DRIVER
    INPUT --> H_INPUT
    OSR --> H_OSR

    style NODE fill:#e1f5ff
    style DRIVER fill:#ffe1e1
    style INPUT fill:#e1ffe1
    style OSR fill:#fff4e1
```

## 데이터 처리 파이프라인

### 패킷 처리 흐름

```mermaid
flowchart TD
    START([패킷 수신]) --> CHECK_HEADER{헤더 확인}
    CHECK_HEADER -->|실패| DROP([패킷 폐기])
    CHECK_HEADER -->|성공| EXTRACT[데이터 추출]

    EXTRACT --> PARSE_AZIMUTH[방위각 파싱]
    PARSE_AZIMUTH --> PARSE_DISTANCE[거리 파싱]
    PARSE_DISTANCE --> PARSE_INTENSITY[강도 파싱]

    PARSE_INTENSITY --> CHECK_RANGE{거리 범위 확인}
    CHECK_RANGE -->|범위 밖| DROP
    CHECK_RANGE -->|범위 내| CHECK_ANGLE{각도 확인}

    CHECK_ANGLE -->|비활성 영역| DROP
    CHECK_ANGLE -->|활성 영역| CONVERT[좌표 변환]

    CONVERT --> POLAR[극좌표 → 직교좌표]
    POLAR --> COMPENSATE{각도 보정?}
    COMPENSATE -->|예| ANGLE_OFFSET[각도 오프셋 적용]
    COMPENSATE -->|아니오| BUFFER
    ANGLE_OFFSET --> BUFFER

    BUFFER --> CHECK_SCAN{스캔 완료?}
    CHECK_SCAN -->|아니오| START
    CHECK_SCAN -->|예| PUBLISH([토픽 발행])

    style START fill:#e1f5ff
    style DROP fill:#ffe1e1
    style PUBLISH fill:#e1ffe1
```

## 네트워크 프로토콜 분석

### MSOP (Measurement Output Packet)

```mermaid
graph LR
    subgraph "MSOP 패킷 구조"
        HEADER[헤더<br/>0xAA 0x55]
        FLAGS[플래그]
        AZIMUTH[방위각]
        DATA[데이터 블록]
        TIMESTAMP[타임스탬프]
        CRC[CRC]
    end

    subgraph "데이터 블록"
        BLOCK_HEADER[블록 헤더]
        POINTS[포인트 데이터<br/>32개]
    end

    subgraph "포인트 데이터"
        DISTANCE[거리<br/>2 bytes]
        INTENSITY[강도<br/>1 byte]
    end

    HEADER --> FLAGS
    FLAGS --> AZIMUTH
    AZIMUTH --> DATA
    DATA --> TIMESTAMP
    TIMESTAMP --> CRC

    DATA --> BLOCK_HEADER
    BLOCK_HEADER --> POINTS
    POINTS --> DISTANCE
    POINTS --> INTENSITY

    style HEADER fill:#e1f5ff
    style DATA fill:#ffe1e1
    style POINTS fill:#e1ffe1
```

### DIFOP (Device Info Output Packet)

```mermaid
graph TB
    subgraph "DIFOP 패킷 구조"
        D_HEADER[헤더<br/>0xA5 0xFF]
        D_TYPE[패킷 타입]
        D_DATA[데이터]
        D_CRC[CRC]
    end

    subgraph "데이터 타입"
        TYPE1[타입 1<br/>기본 정보]
        TYPE2[타입 2<br/>온도/RPM]
        TYPE3[타입 3<br/>각도 보정]
    end

    D_HEADER --> D_TYPE
    D_TYPE --> D_DATA
    D_DATA --> D_CRC

    D_TYPE --> TYPE1
    D_TYPE --> TYPE2
    D_TYPE --> TYPE3

    style D_HEADER fill:#e1f5ff
    style D_DATA fill:#ffe1e1
```

## 좌표 변환 수학

### 극좌표에서 직교좌표로

```mermaid
graph TB
    subgraph "입력"
        R[거리 r]
        THETA[방위각 θ]
        PHI[고도각 φ]
    end

    subgraph "변환"
        X[x = r × cos(φ) × sin(θ)]
        Y[y = r × cos(φ) × cos(θ)]
        Z[z = r × sin(φ)]
    end

    subgraph "출력"
        POINT[PointXYZIT]
    end

    R --> X
    R --> Y
    R --> Z
    THETA --> X
    THETA --> Y
    PHI --> X
    PHI --> Y
    PHI --> Z

    X --> POINT
    Y --> POINT
    Z --> POINT

    style R fill:#e1f5ff
    style THETA fill:#ffe1e1
    style PHI fill:#e1ffe1
    style POINT fill:#fff4e1
```

## 시리얼 통신 분석

### 시리얼 포트 설정

```mermaid
graph TB
    subgraph "시리얼 파라미터"
        BAUD[보레이트<br/>460800]
        BITS[데이터 비트<br/>8]
        PARITY[패리티<br/>None]
        STOP[스톱 비트<br/>1]
    end

    subgraph "termios 설정"
        CFLAG[c_cflag]
        IFLAG[c_iflag]
        CC[c_cc]
    end

    subgraph "설정 값"
        CLOCAL[CLOCAL<br/>모뎀 제어 무시]
        CREAD[CREAD<br/>수신 가능]
        CS8[CS8<br/>8비트]
        VTIME[VTIME<br/>타임아웃]
        VMIN[VMIN<br/>최문 문자]
    end

    BAUD --> CFLAG
    BITS --> CFLAG
    PARITY --> CFLAG
    STOP --> CFLAG

    CFLAG --> CLOCAL
    CFLAG --> CREAD
    CFLAG --> CS8
    CFLAG --> CC

    CC --> VTIME
    CC --> VMIN

    style BAUD fill:#e1f5ff
    style CFLAG fill:#ffe1e1
    style CLOCAL fill:#e1ffe1
```

## 스레드 동기화 메커니즘

```mermaid
sequenceDiagram
    participant R as 수신 스레드
    participant M as Mutex
    participant B as 버퍼
    participant C as Condition Variable
    participant P as 발행 스레드

    R->>M: lock()
    R->>B: 데이터 쓰기
    R->>M: unlock()
    R->>C: notify_one()

    P->>C: wait()
    P->>M: lock()
    P->>B: 데이터 읽기
    P->>M: unlock()
    P->>P: 데이터 처리

    Note over R,P: 생산자-소비자 패턴
```

## 진단 시스템 상세

### 진단 항목 및 임계값

```mermaid
graph TB
    subgraph "진단 카테고리"
        HARDWARE[하드웨어 상태]
        COMMUNICATION[통신 상태]
        DATA[데이터 품질]
    end

    subgraph "하드웨어 상태"
        TEMP[온도<br/>-20°C to 60°C]
        RPM[RPM<br/>600 to 1200]
        POWER[전압<br/>9V to 32V]
    end

    subgraph "통신 상태"
        LINK[링크 상태<br/>Connected/Disconnected]
        PACKET_RATE[패킷 수신률<br/>10-20 Hz]
        PACKET_LOSS[패킷 손실률<br/>< 1%]
    end

    subgraph "데이터 품질"
        POINT_COUNT[포인트 수<br/>모델별 상이]
        INTENSITY[강도 범위<br/>0-255]
        DISTANCE[거리 범위<br/>0-200m]
    end

    HARDWARE --> TEMP
    HARDWARE --> RPM
    HARDWARE --> POWER

    COMMUNICATION --> LINK
    COMMUNICATION --> PACKET_RATE
    COMMUNICATION --> PACKET_LOSS

    DATA --> POINT_COUNT
    DATA --> INTENSITY
    DATA --> DISTANCE

    style HARDWARE fill:#e1f5ff
    style COMMUNICATION fill:#ffe1e1
    style DATA fill:#e1ffe1
```

## 파라미터 상세 분석

### 필수 파라미터

| 파라미터 | 타입 | 기본값 | 설명 |
|---------|------|--------|------|
| `device_ip` | string | 192.168.1.200 | 라이다 IP 주소 |
| `msop_port` | int | 2368 | MSOP 데이터 포트 |
| `lidar_name` | string | M10 | 라이다 모델 |
| `frame_id` | string | laser_link | 좌표 프레임 ID |

### 선택적 파라미터

| 파라미터 | 타입 | 기본값 | 설명 |
|---------|------|--------|------|
| `min_range` | double | 0.0 | 최소 거리 (m) |
| `max_range` | double | 200.0 | 최대 거리 (m) |
| `angle_disable_min` | double | 0.0 | 비활성 각도 시작 (도) |
| `angle_disable_max` | double | 0.0 | 비활성 각도 끝 (도) |
| `use_gps_ts` | bool | false | GPS 타임스탬프 사용 |
| `compensation` | bool | false | 각도 보정 사용 |

## 성능 특성

### 시스템 성능

```mermaid
graph TB
    subgraph "성능 지표"
        LATENCY[지연 시간<br/>less than 50ms]
        THROUGHPUT[처리량<br/>about 400K points/s]
        CPU[CPU 사용량<br/>less than 20%]
        MEMORY[메모리 사용량<br/>less than 100MB]
    end

    subgraph "네트워크"
        BANDWIDTH[대역폭<br/>about 10 Mbps]
        PACKET_SIZE[패킷 크기<br/>about 1200 bytes]
        PACKET_RATE[패킷률<br/>10-20 Hz]
    end

    subgraph "출력"
        SCAN_RATE[스캔률<br/>10-20 Hz]
        POINTS_PER_SCAN[포인트/스캔<br/>about 20000]
        TOPIC_RATE[토픽률<br/>10-20 Hz]
    end

    LATENCY --> THROUGHPUT
    THROUGHPUT --> CPU
    CPU --> MEMORY

    BANDWIDTH --> PACKET_SIZE
    PACKET_SIZE --> PACKET_RATE

    SCAN_RATE --> POINTS_PER_SCAN
    POINTS_PER_SCAN --> TOPIC_RATE

    style LATENCY fill:#e1f5ff
    style BANDWIDTH fill:#ffe1e1
    style SCAN_RATE fill:#e1ffe1
```

## 트러블슈팅 가이드

### 일반적인 문제 및 해결책

```mermaid
flowchart TD
    START([문제 발생]) --> CHECK_TYPE{문제 유형}

    CHECK_TYPE -->|연결 불가| CONN[연결 문제 진단]
    CHECK_TYPE -->|데이터 없음| DATA[데이터 문제 진단]
    CHECK_TYPE -->|성능 저하| PERF[성능 문제 진단]

    CONN --> CHECK_IP{IP 확인}
    CHECK_IP -->|불일치| FIX_IP[IP 수정]
    CHECK_IP -->|일치| CHECK_PORT{포트 확인}
    CHECK_PORT -->|불일치| FIX_PORT[포트 수정]
    CHECK_PORT -->|일치| CHECK_FIREWALL[방화벽 확인]

    DATA --> CHECK_LIDAR{라이다 상태}
    CHECK_LIDAR -->|꺼짐| POWER_ON[전원 켜기]
    CHECK_LIDAR -->|켜짐| CHECK_CABLE[케이블 확인]

    PERF --> CHECK_CPU{CPU 사용량}
    CHECK_CPU -->|높음| REDUCE_LOAD[부하 감소]
    CHECK_CPU -->|정상| CHECK_NETWORK[네트워크 확인]

    FIX_IP --> RESOLVED([해결됨])
    FIX_PORT --> RESOLVED
    CHECK_FIREWALL --> RESOLVED
    POWER_ON --> RESOLVED
    CHECK_CABLE --> RESOLVED
    REDUCE_LOAD --> RESOLVED
    CHECK_NETWORK --> RESOLVED

    style START fill:#e1f5ff
    style RESOLVED fill:#e1ffe1
```

## 확장 가능성

### 플러그인 아키텍처

```mermaid
graph TB
    subgraph "코어 드라이버"
        CORE[LslidarDriver]
        INPUT_BASE[Input]
    end

    subgraph "플러그인"
        SOCKET[InputSocket]
        PCAP[InputPCAP]
        CUSTOM[CustomInput]
    end

    subgraph "확장 포인트"
        DATA_SOURCE[데이터 소스]
        PARSER[파서]
        FILTER[필터]
        OUTPUT[출력]
    end

    CORE --> INPUT_BASE
    INPUT_BASE <|-- SOCKET
    INPUT_BASE <|-- PCAP
    INPUT_BASE <|-- CUSTOM

    CORE --> DATA_SOURCE
    CORE --> PARSER
    CORE --> FILTER
    CORE --> OUTPUT

    style CORE fill:#e1f5ff
    style SOCKET fill:#ffe1e1
    style PCAP fill:#e1ffe1
    style CUSTOM fill:#fff4e1
```

## 테스트 전략

### 테스트 계층

```mermaid
graph TB
    subgraph "단위 테스트"
        UNIT_INPUT[Input 클래스]
        UNIT_OSR[LSIOSR 클래스]
        UNIT_PARSER[파서 함수]
    end

    subgraph "통합 테스트"
        INT_DRIVER[드라이버 통합]
        INT_NETWORK[네트워크 통신]
        INT_SERIAL[시리얼 통신]
    end

    subgraph "시스템 테스트"
        SYS_E2E[엔드 투 엔드]
        SYS_PERF[성능 테스트]
        SYS_STRESS[스트레스 테스트]
    end

    UNIT_INPUT --> INT_DRIVER
    UNIT_OSR --> INT_DRIVER
    UNIT_PARSER --> INT_DRIVER

    INT_DRIVER --> SYS_E2E
    INT_NETWORK --> SYS_E2E
    INT_SERIAL --> SYS_E2E

    SYS_E2E --> SYS_PERF
    SYS_PERF --> SYS_STRESS

    style UNIT_INPUT fill:#e1f5ff
    style INT_DRIVER fill:#ffe1e1
    style SYS_E2E fill:#e1ffe1
```

## 보안 고려사항

### 보안 위험 및 완화

```mermaid
graph TB
    subgraph "보안 위험"
        R1[네트워크 스니핑]
        R2[데이터 변조]
        R3[서비스 거부]
        R4[권한 상승]
    end

    subgraph "완화 조치"
        M1[네트워크 격리]
        M2[CRC 검증]
        M3[속도 제한]
        M4[최소 권한]
    end

    R1 --> M1
    R2 --> M2
    R3 --> M3
    R4 --> M4

    style R1 fill:#ffe1e1
    style M1 fill:#e1ffe1
```

## 향후 개선 방향

### 제안된 개선사항

```mermaid
graph TB
    subgraph "단기 개선"
        S1[문서화 개선]
        S2[테스트 커버리지]
        S3[코드 리팩토링]
    end

    subgraph "중기 개선"
        M1[추가 라이다 지원]
        M2[성능 최적화]
        M3[ROS2 최신 버전 지원]
    end

    subgraph "장기 개선"
        L1[GPU 가속]
        L2[실시간 처리]
        L3[클라우드 통합]
    end

    S1 --> M1
    S2 --> M2
    S3 --> M3

    M1 --> L1
    M2 --> L2
    M3 --> L3

    style S1 fill:#e1f5ff
    style M1 fill:#ffe1e1
    style L1 fill:#e1ffe1
```

## 참고 자료

- ROS2 문서: https://docs.ros.org/
- Leishen 라이다 문서: honghangli@lslidar.com
- PCL 라이브러리: https://pointclouds.org/
- diagnostic_updater: https://index.ros.org/p/diagnostic_updater/