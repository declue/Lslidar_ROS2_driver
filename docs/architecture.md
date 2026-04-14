# Lslidar ROS2 Driver 아키텍처 문서

## 개요

Lslidar ROS2 Driver는 Leishen(雷神) M10, M10_GPS, M10_P, M10_PLUS, N10 시리즈 라이다를 위한 ROS2 드라이버입니다. Ubuntu 20.04와 ROS2 FOXY에서 테스트되었습니다.

## 시스템 아키텍처

```mermaid
graph TB
    subgraph "하드웨어 계층"
        LIDAR[Leishen LIDAR<br/>M10/N10 시리즈]
        SERIAL[시리얼 포트<br/>/dev/ttyUSB0]
        NETWORK[네트워크<br/>UDP 2368/2369]
    end

    subgraph "ROS2 노드 계층"
        DRIVER[lslidar_driver_node<br/>LifecycleNode]
        RVIZ[rviz2<br/>시각화]
    end

    subgraph "드라이버 내부 구성요소"
        INPUT[Input<br/>데이터 수집]
        OSR[LSIOSR<br/>시리얼 통신]
        PROCESSOR[데이터 처리<br/>파싱/변환]
        PUBLISHER[토픽 발행]
    end

    subgraph "ROS2 토픽/서비스"
        SCAN[/scan<br/>LaserScan]
        POINTCLOUD[/lslidar_point_cloud<br/>PointCloud2]
        ORDER[/lslidar_order<br/>Int8]
        DIAGNOSTICS[/diagnostics<br/>diagnostic_msgs]
    end

    LIDAR -->|UDP| NETWORK
    LIDAR -->|시리얼| SERIAL
    NETWORK --> INPUT
    SERIAL --> OSR
    INPUT --> PROCESSOR
    OSR --> PROCESSOR
    PROCESSOR --> PUBLISHER
    PUBLISHER --> SCAN
    PUBLISHER --> POINTCLOUD
    ORDER --> DRIVER
    SCAN --> RVIZ
    POINTCLOUD --> RVIZ
    DRIVER --> DIAGNOSTICS

    style DRIVER fill:#e1f5ff
    style LIDAR fill:#ffe1e1
    style RVIZ fill:#e1ffe1
```

## 패키지 구조

```mermaid
graph LR
    subgraph "lslidar_msgs 패키지"
        MSG1[LslidarPacket.msg]
        MSG2[LslidarScan.msg]
        MSG3[LslidarPoint.msg]
        MSG4[LslidarSweep.msg]
        MSG5[LslidarDifop.msg]
    end

    subgraph "lslidar_driver 패키지"
        NODE[lslidar_driver_node]
        LAUNCH[launch/]
        PARAMS[params/]
        RVIZ[rviz/]
        SRC[src/]
        INCLUDE[include/]
    end

    MSG1 -.->|사용| NODE
    MSG2 -.->|사용| NODE
    MSG3 -.->|사용| NODE
    MSG4 -.->|사용| NODE
    MSG5 -.->|사용| NODE

    style MSG1 fill:#fff4e1
    style MSG2 fill:#fff4e1
    style MSG3 fill:#fff4e1
    style MSG4 fill:#fff4e1
    style MSG5 fill:#fff4e1
```

## 데이터 흐름

```mermaid
sequenceDiagram
    participant L as LIDAR 장치
    participant N as lslidar_driver_node
    participant I as Input (UDP/PCAP)
    participant S as LSIOSR (Serial)
    participant P as 데이터 처리
    participant R as ROS2 토픽
    participant V as rviz2

    L->>I: UDP 패킷 (MSOP)
    L->>S: 시리얼 데이터 (DIFOP)
    I->>N: getPacket()
    S->>N: read()/send()
    N->>P: data_processing()
    P->>P: 패킷 파싱
    P->>P: 좌표 변환
    P->>P: 필터링
    P->>R: LaserScan 발행
    P->>R: PointCloud2 발행
    R->>V: 시각화 데이터
    V->>V: 렌더링

    Note over N,R: 주기: 10-20Hz (RPM에 따라)
```

## 클래스 다이어그램

```mermaid
classDiagram
    class LslidarDriver {
        +LslidarDriver()
        +~LslidarDriver()
        +initialize() bool
        +polling() bool
        -loadParameters() bool
        -createRosIO() bool
        -open_serial()
        -data_processing()
        -pubScanThread()
        -recvThread_crc()
        -getScan()
        -scan_points_ vector&lt;ScanPoint&gt;
        -msop_input_ Input*
        -serial_ LSIOSR*
        -scan_pub Publisher
        -point_cloud_pub Publisher
    }

    class Input {
        <<abstract>>
        +Input(Node*, uint16_t)
        +getPacket() int
        +getRpm() int
        +getReturnMode() int
        +UDP_order()
        +UDP_difop()
        #private_nh_ Node*
        #port_ uint16_t
        #cur_rpm_ int
        #return_mode_ int
    }

    class InputSocket {
        +InputSocket(Node*, uint16_t)
        +~InputSocket()
        +getPacket() int
        -devip_ in_addr
        -devip_difop in_addr
    }

    class InputPCAP {
        +InputPCAP(Node*, uint16_t, double, string)
        +~InputPCAP()
        +getPacket() int
        -pcap_ pcap_t_ptr
        -filename_ string
        -packet_rate_ Rate
    }

    class LSIOSR {
        +instance(string, int, int) LSIOSR*
        +read() int
        +send() int
        +flushinput()
        +init() int
        +close() int
        -port_ string
        -baud_rate_ int
        -fd_ int
        -setOpt() int
        -waitReadable() int
        -waitWritable() int
    }

    class ScanPoint {
        +degree double
        +range double
        +intensity double
    }

    class PointXYZIT {
        +x float
        +y float
        +z float
        +intensity uint8_t
        +timestamp double
    }

    LslidarDriver --> Input : uses
    LslidarDriver --> LSIOSR : uses
    LslidarDriver --> ScanPoint : contains
    LslidarDriver --> PointXYZIT : uses
    Input <|-- InputSocket
    Input <|-- InputPCAP
```

## 메시지 구조

```mermaid
graph TB
    subgraph "LslidarPacket"
        PKT_STAMP[stamp: Time]
        PKT_DATA[data: uint8[2000]]
    end

    subgraph "LslidarPoint"
        PT_TIME[time: float32]
        PT_X[x: float64]
        PT_Y[y: float64]
        PT_Z[z: float64]
        PT_AZIMUTH[azimuth: float64]
        PT_DISTANCE[distance: float64]
        PT_INTENSITY[intensity: float64]
    end

    subgraph "LslidarScan"
        SC_ALT[altitude: float64]
        SC_POINTS[points: LslidarPoint[]]
    end

    subgraph "LslidarSweep"
        SW_HEADER[header: std_msgs/Header]
        SW_SCANS[scans: LslidarScan[16]]
    end

    subgraph "LslidarDifop"
        DF_TEMP[temperature: int64]
        DF_RPM[rpm: int64]
    end

    SW_SCANS --> SC_POINTS
    SC_POINTS --> PT_TIME
    SC_POINTS --> PT_X
    SC_POINTS --> PT_Y
    SC_POINTS --> PT_Z

    style PKT_STAMP fill:#e1f5ff
    style PT_TIME fill:#e1f5ff
    style SC_ALT fill:#e1f5ff
    style SW_HEADER fill:#e1f5ff
```

## 네트워크 통신 구조

```mermaid
graph LR
    subgraph "LIDAR 장치"
        LIDAR_IP[192.168.1.200<br/>MSOP 데이터]
        LIDAR_DIFOP[192.168.1.102<br/>DIFOP 데이터]
    end

    subgraph "PC/로봇"
        PC_IP[192.168.1.x]
        MSOP_PORT[UDP 2368<br/>MSOP 수신]
        DIFOP_PORT[UDP 2369<br/>DIFOP 수신]
        SERIAL[/dev/ttyUSB0<br/>시리얼 통신]
    end

    subgraph "멀티캐스트 옵션"
        GROUP[224.1.1.2<br/>멀티캐스트 그룹]
    end

    LIDAR_IP -->|UDP| MSOP_PORT
    LIDAR_DIFOP -->|UDP| DIFOP_PORT
    LIDAR_DIFOP -->|시리얼| SERIAL

    MSOP_PORT -.->|선택적| GROUP
    DIFOP_PORT -.->|선택적| GROUP

    style LIDAR_IP fill:#ffe1e1
    style LIDAR_DIFOP fill:#ffe1e1
    style PC_IP fill:#e1ffe1
```

## 스레드 구조

```mermaid
graph TB
    subgraph "메인 스레드"
        MAIN[메인 루프<br/>polling()]
        INIT[initialize()]
        PARAM[loadParameters()]
        IO[createRosIO()]
    end

    subgraph "수신 스레드"
        RECV[recvThread_crc()]
        PACKET[getPacket()]
        CRC[CRC 검사]
    end

    subgraph "발행 스레드"
        PUB[pubScanThread()]
        SCAN[getScan()]
        CONVERT[좌표 변환]
        PUBLISH[토픽 발행]
    end

    subgraph "동기화"
        MUTEX[mutex_]
        COND[pubscan_cond_]
    end

    MAIN --> INIT
    INIT --> PARAM
    PARAM --> IO
    IO --> RECV
    IO --> PUB

    RECV --> PACKET
    PACKET --> CRC
    CRC --> MUTEX

    PUB --> SCAN
    SCAN --> CONVERT
    CONVERT --> PUBLISH
    PUBLISH --> MUTEX

    MUTEX --> COND
    COND --> PUB

    style MAIN fill:#e1f5ff
    style RECV fill:#ffe1e1
    style PUB fill:#e1ffe1
```

## 상태 전이 다이어그램

```mermaid
stateDiagram-v2
    [*] --> UNCONFIGURED: 노드 생성
    UNCONFIGURED --> INACTIVE: configure()
    INACTIVE --> ACTIVE: activate()
    ACTIVE --> INACTIVE: deactivate()
    INACTIVE --> UNCONFIGURED: cleanup()
    ACTIVE --> FINALIZE: shutdown()
    INACTIVE --> FINALIZE: shutdown()
    UNCONFIGURED --> FINALIZE: shutdown()
    FINALIZE --> [*]

    state ACTIVE {
        [*] --> RECEIVING: 데이터 수신 시작
        RECEIVING --> PROCESSING: 패킷 수신
        PROCESSING --> PUBLISHING: 데이터 처리
        PUBLISHING --> RECEIVING: 토픽 발행 완료
    }

    note right of ACTIVE
        라이다 데이터 수집 및
        토픽 발행 활성 상태
    end note
```

## 타이밍 다이어그램

```mermaid
gantt
    title 라이다 데이터 처리 타이밍
    dateFormat s
    axisFormat %S.%L

    section UDP 수신
    패킷 수신           :0, 0.05
    CRC 검사            :0.05, 0.01

    section 데이터 처리
    패킷 파싱           :0.06, 0.02
    좌표 변환           :0.08, 0.03
    필터링              :0.11, 0.01

    section 토픽 발행
    LaserScan 발행      :0.12, 0.01
    PointCloud2 발행    :0.13, 0.02

    section 주기
    다음 스캔 시작       :0.15, 0.05
```

## 배포 다이어그램

```mermaid
graph TB
    subgraph "ROS2 워크스페이스"
        WS[workspace/]
        SRC[src/]
        BUILD[build/]
        INSTALL[install/]
        LOG[log/]
    end

    subgraph "lslidar_msgs"
        MSG_PKG[lslidar_msgs/]
        MSG_SRC[msg/]
        MSG_CMAKE[CMakeLists.txt]
        MSG_PKGXML[package.xml]
    end

    subgraph "lslidar_driver"
        DRV_PKG[lslidar_driver/]
        DRV_SRC[src/]
        DRV_INC[include/]
        DRV_LAUNCH[launch/]
        DRV_PARAMS[params/]
        DRV_RVIZ[rviz/]
        DRV_CMAKE[CMakeLists.txt]
        DRV_PKGXML[package.xml]
    end

    WS --> SRC
    WS --> BUILD
    WS --> INSTALL
    WS --> LOG

    SRC --> MSG_PKG
    SRC --> DRV_PKG

    MSG_PKG --> MSG_SRC
    MSG_PKG --> MSG_CMAKE
    MSG_PKG --> MSG_PKGXML

    DRV_PKG --> DRV_SRC
    DRV_PKG --> DRV_INC
    DRV_PKG --> DRV_LAUNCH
    DRV_PKG --> DRV_PARAMS
    DRV_PKG --> DRV_RVIZ
    DRV_PKG --> DRV_CMAKE
    DRV_PKG --> DRV_PKGXML

    style WS fill:#e1f5ff
    style MSG_PKG fill:#ffe1e1
    style DRV_PKG fill:#e1ffe1
```

## 파라미터 구조

```mermaid
graph TB
    subgraph "네트워크 파라미터"
        DEV_IP[device_ip: 192.168.1.200]
        DEV_DIFOP[device_ip_difop: 192.168.1.102]
        MSOP_PORT[msop_port: 2368]
        DIFOP_PORT[difop_port: 2369]
        GROUP_IP[group_ip: 224.1.1.2]
        MULTICAST[add_multicast: false]
    end

    subgraph "시리얼 파라미터"
        SERIAL_PORT[serial_port_: /dev/ttyUSB0]
        INTERFACE[interface_selection: serial]
    end

    subgraph "라이다 파라미터"
        LIDAR_NAME[lidar_name: M10]
        FRAME_ID[frame_id: laser_link]
        HIGH_REF[high_reflection: false]
        COMPENSATION[compensation: false]
    end

    subgraph "데이터 파라미터"
        MIN_RANGE[min_range: 0.0]
        MAX_RANGE[max_range: 200.0]
        ANGLE_MIN[angle_disable_min: 0.0]
        ANGLE_MAX[angle_disable_max: 0.0]
        USE_GPS[use_gps_ts: false]
    end

    subgraph "토픽 파라미터"
        SCAN_TOPIC[scan_topic: /scan]
        PC_TOPIC[pointcloud_topic: /lslidar_point_cloud]
        PUB_SCAN[pubScan: true]
        PUB_PC[pubPointCloud2: false]
    end

    style DEV_IP fill:#e1f5ff
    style SERIAL_PORT fill:#ffe1e1
    style LIDAR_NAME fill:#e1ffe1
    style MIN_RANGE fill:#fff4e1
    style SCAN_TOPIC fill:#f4e1ff
```

## 주요 기능 흐름

```mermaid
flowchart TD
    START([시작]) --> INIT[노드 초기화]
    INIT --> LOAD[파라미터 로드]
    LOAD --> CHECK_INTERFACE{인터페이스 선택}

    CHECK_INTERFACE -->|serial| OPEN_SERIAL[시리얼 포트 열기]
    CHECK_INTERFACE -->|net| OPEN_SOCKET[UDP 소켓 열기]

    OPEN_SERIAL --> WAIT_DATA[데이터 대기]
    OPEN_SOCKET --> WAIT_DATA

    WAIT_DATA --> CHECK_DATA{데이터 수신?}
    CHECK_DATA -->|아니오| WAIT_DATA
    CHECK_DATA -->|예| PARSE[패킷 파싱]

    PARSE --> CRC_CHECK{CRC 검사}
    CRC_CHECK -->|실패| WAIT_DATA
    CRC_CHECK -->|성공| PROCESS[데이터 처리]

    PROCESS --> CONVERT[좌표 변환]
    CONVERT --> FILTER[필터링]
    FILTER --> COMPENSATE{각도 보정?}

    COMPENSATE -->|예| ANGLE_COMP[각도 보정]
    COMPENSATE -->|아니오| PUBLISH
    ANGLE_COMP --> PUBLISH

    PUBLISH --> CHECK_PUB{토픽 발행?}
    CHECK_PUB -->|scan| PUB_SCAN[LaserScan 발행]
    CHECK_PUB -->|pointcloud| PUB_PC[PointCloud2 발행]
    CHECK_PUB -->|둘 다| PUB_SCAN
    PUB_SCAN --> DIAG[진단 업데이트]
    PUB_PC --> DIAG

    DIAG --> CONTINUE{계속?}
    CONTINUE -->|예| WAIT_DATA
    CONTINUE -->|아니오| END([종료])

    style START fill:#e1f5ff
    style END fill:#ffe1e1
    style INIT fill:#e1ffe1
    style PUBLISH fill:#fff4e1
```

## 지원 라이다 모델

```mermaid
graph LR
    subgraph "M10 시리즈"
        M10[M10]
        M10_GPS[M10_GPS]
        M10_P[M10_P]
        M10_PLUS[M10_PLUS]
    end

    subgraph "N10 시리즈"
        N10[N10]
        N10_P[N10_P]
    end

    subgraph "기타"
        L10[L10]
    end

    ALL[Leishen LIDAR] --> M10
    ALL --> N10
    ALL --> L10

    style M10 fill:#e1f5ff
    style N10 fill:#ffe1e1
    style L10 fill:#e1ffe1
```

## 진단 및 모니터링

```mermaid
graph TB
    subgraph "진단 시스템"
        UPDATER[diagnostic_updater::Updater]
        TOPIC_DIAG[TopicDiagnostic]
        MIN_FREQ[diag_min_freq]
        MAX_FREQ[diag_max_freq]
    end

    subgraph "진단 항목"
        LINK[링크 상태]
        DATA_RATE[데이터 수신률]
        PACKET_LOSS[패킷 손실]
        TEMPERATURE[온도]
        RPM[RPM]
    end

    UPDATER --> TOPIC_DIAG
    TOPIC_DIAG --> MIN_FREQ
    TOPIC_DIAG --> MAX_FREQ

    UPDATER --> LINK
    UPDATER --> DATA_RATE
    UPDATER --> PACKET_LOSS
    UPDATER --> TEMPERATURE
    UPDATER --> RPM

    style UPDATER fill:#e1f5ff
    style LINK fill:#ffe1e1
    style DATA_RATE fill:#e1ffe1
```

## 빌드 시스템

```mermaid
graph TB
    subgraph "빌드 프로세스"
        COLCON[colcon build]
        MSG_BUILD[lslidar_msgs 빌드]
        DRV_BUILD[lslidar_driver 빌드]
    end

    subgraph "lslidar_msgs 의존성"
        MSG_DEP1[ament_cmake]
        MSG_DEP2[std_msgs]
        MSG_DEP3[sensor_msgs]
        MSG_DEP4[builtin_interfaces]
        MSG_DEP5[rosidl_default_generators]
    end

    subgraph "lslidar_driver 의존성"
        DRV_DEP1[rclcpp]
        DRV_DEP2[std_msgs]
        DRV_DEP3[lslidar_msgs]
        DRV_DEP4[pcl_conversions]
        DRV_DEP5[rclpy]
        DRV_DEP6[libpcap]
        DRV_DEP7[libpcl-all]
        DRV_DEP8[pluginlib]
        DRV_DEP9[sensor_msgs]
        DRV_DEP10[diagnostic_updater]
        DRV_DEP11[Boost]
    end

    COLCON --> MSG_BUILD
    COLCON --> DRV_BUILD

    MSG_BUILD --> MSG_DEP1
    MSG_BUILD --> MSG_DEP2
    MSG_BUILD --> MSG_DEP3
    MSG_BUILD --> MSG_DEP4
    MSG_BUILD --> MSG_DEP5

    DRV_BUILD --> DRV_DEP1
    DRV_BUILD --> DRV_DEP2
    DRV_BUILD --> DRV_DEP3
    DRV_BUILD --> DRV_DEP4
    DRV_BUILD --> DRV_DEP5
    DRV_BUILD --> DRV_DEP6
    DRV_BUILD --> DRV_DEP7
    DRV_BUILD --> DRV_DEP8
    DRV_BUILD --> DRV_DEP9
    DRV_BUILD --> DRV_DEP10
    DRV_BUILD --> DRV_DEP11

    style COLCON fill:#e1f5ff
    style MSG_BUILD fill:#ffe1e1
    style DRV_BUILD fill:#e1ffe1
```

## 요약

이 Lslidar ROS2 Driver는 다음과 같은 특징을 가집니다:

1. **다중 인터페이스 지원**: UDP 네트워크 및 시리얼 포트 통신
2. **다양한 라이다 모델**: M10, N10 시리즈 지원
3. **PCAP 재생**: 네트워크 패킷 캡처 파일 재생 지원
4. **다중 출력 형식**: LaserScan 및 PointCloud2 토픽 지원
5. **진단 시스템**: diagnostic_updater를 통한 상태 모니터링
6. **각도 보정**: M10 시리즈 각도 보정 기능
7. **멀티캐스트 지원**: 네트워크 멀티캐스트 옵션
8. **GPS 타임스탬프**: GPS 동기화 옵션