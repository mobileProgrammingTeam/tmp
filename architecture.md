작업자 안전 모니터링 시스템 아키텍처

1. 전체 아키텍처

flowchart LR
    subgraph Worker["작업자"]
        ACC["가속도 센서"]
        BEACON["BLE Beacon"]
        WATCH["Smart Watch<br/>HR / SpO₂"]

        subgraph APP["Android App - Kotlin"]
            FD["FallDetector"]
            BD["BeaconDetector"]
            VD["VitalDetector"]
            SM["SafetyManager"]
        end

        ACC --> FD
        BEACON --> BD
        WATCH --> VD

        FD --> SM
        BD --> SM
        VD --> SM
    end

    subgraph Backend["Backend"]
        API["FastAPI<br/>Google Cloud Run"]
        DB[("Supabase PostgreSQL")]
        RT["Supabase Realtime"]

        API --> DB
        DB --> RT
    end

    subgraph Admin["관리자"]
        WEB["Admin Web<br/>HTML / CSS / Vanilla JavaScript"]
    end

    SM -->|"HTTPS / REST"| API
    WEB -->|"REST"| API
    RT -->|"Realtime"| WEB

전체 데이터 흐름은 다음과 같다.

Sensor / IoT → Kotlin Detector → SafetyManager → FastAPI → Supabase → 관리자 Web

---

2. Android 감지 모듈

낙상 감지, 구역 접근 감지, 생체신호 감지는 사용하는 센서와 판단 방식이 서로 다르므로 독립적인 클래스로 구현한다.

세 기능을 애플리케이션에서 동일한 방식으로 관리하기 위해 공통 "SafetyDetector" 인터페이스를 정의한다.

classDiagram
    class SafetyDetector {
        <<interface>>
        +DetectorType detectorType
        +start()
        +stop()
        +setEventListener(listener)
    }

    class FallDetector
    class BeaconDetector
    class VitalDetector
    class SafetyManager

    SafetyDetector <|.. FallDetector
    SafetyDetector <|.. BeaconDetector
    SafetyDetector <|.. VitalDetector

    SafetyManager --> SafetyDetector

SafetyDetector

interface SafetyDetector {

    val detectorType: DetectorType

    fun start()

    fun stop()

    fun setEventListener(
        listener: (SafetyEvent) -> Unit
    )
}

각 구성요소의 역할은 다음과 같다.

구성| 역할
"detectorType"| 감지 모듈 종류 식별
"start()"| 센서 또는 IoT 장치 모니터링 시작
"stop()"| 모니터링 종료 및 자원 해제
"setEventListener()"| 위험 상황 발생 시 "SafetyEvent" 전달

Detector 종류는 공통 "enum"으로 정의한다.

enum class DetectorType {
    FALL,
    BEACON,
    VITAL
}

각 기능 담당자는 "SafetyDetector"를 구현한다.

class FallDetector : SafetyDetector {
    // 가속도 센서 및 낙상 판단 구현
}

class BeaconDetector : SafetyDetector {
    // BLE Beacon 스캔 및 접근 판단 구현
}

class VitalDetector : SafetyDetector {
    // 심박수 / SpO₂ 수집 및 이상 판단 구현
}

각 Detector의 센서 접근 방식과 판단 알고리즘은 독립적으로 구현하되, 외부에서는 모두 "SafetyDetector" 타입으로 취급할 수 있도록 한다.

---

3. SafetyEvent

모든 Detector는 위험 상황을 감지하면 공통 형식인 "SafetyEvent"를 생성한다.

data class SafetyEvent(
    val type: DetectorType,
    val severity: Severity,
    val message: String,
    val timestamp: Long = System.currentTimeMillis()
)

위험도 역시 공통 타입으로 정의한다.

enum class Severity {
    LOW,
    MEDIUM,
    HIGH,
    CRITICAL
}

예를 들어 낙상이 감지되면 "FallDetector"는 다음과 같이 이벤트를 발생시킨다.

listener?.invoke(
    SafetyEvent(
        type = DetectorType.FALL,
        severity = Severity.HIGH,
        message = "낙상 감지"
    )
)

따라서 세 감지 기능의 내부 구현이 서로 달라도 최종 출력은 모두 "SafetyEvent"로 통일된다.

---

4. SafetyManager

"SafetyManager"는 모든 "SafetyDetector"를 등록하고 관리하는 상위 모듈이다.

class SafetyManager(
    private val detectors: List<SafetyDetector>
) {

    fun startMonitoring() {
        detectors.forEach { detector ->

            detector.setEventListener { event ->
                handleEvent(event)
            }

            detector.start()
        }
    }

    fun stopMonitoring() {
        detectors.forEach {
            it.stop()
        }
    }

    private fun handleEvent(event: SafetyEvent) {
        // 앱 내부 위험 처리
        // 서버 전송
    }
}

Detector 등록 예시는 다음과 같다.

val safetyManager = SafetyManager(
    listOf(
        FallDetector(),
        BeaconDetector(),
        VitalDetector()
    )
)

safetyManager.startMonitoring()

각 기능 담당자가 지켜야 할 공통 규칙은 다음과 같다.

«"SafetyDetector" 인터페이스를 구현하고, 위험 상황을 감지하면 "SafetyEvent"를 발생시킨다.»

이를 통해 낙상, Beacon, 생체신호 담당자가 각 기능을 독립적으로 개발하면서도 하나의 Android 애플리케이션으로 통합할 수 있다.

---

5. 백엔드 및 관리자 인터페이스

Backend

FastAPI + Google Cloud Run을 사용한다.

주요 역할은 다음과 같다.

- Android 애플리케이션의 REST API 요청 수신
- 요청 데이터 검증
- 위험 이벤트 처리
- Supabase 데이터 저장 및 조회
- 관리자 인터페이스용 API 제공

Supabase

다음 기능을 담당한다.

- PostgreSQL 기반 데이터 저장
- 사용자 인증
- Realtime 기반 실시간 이벤트 전달

관리자 Web

HTML / CSS / Vanilla JavaScript를 사용한다.

주요 기능은 다음과 같다.

- 작업자 상태 확인
- 심박수 및 SpO₂ 확인
- 위험 이벤트 실시간 확인
- 위험 이벤트 이력 조회

위험 이벤트의 최종 처리 흐름은 다음과 같다.

"Sensor → SafetyDetector → SafetyEvent → SafetyManager → FastAPI → Supabase → Admin Web"

---

6. 기술 스택

영역| 기술
Mobile| Android / Kotlin
낙상 감지| Accelerometer
구역 접근 감지| BLE Beacon
생체신호| Smart Watch / Heart Rate / SpO₂
Backend| Python / FastAPI
Backend Hosting| Google Cloud Run
Database| Supabase PostgreSQL
Authentication| Supabase Auth
Realtime| Supabase Realtime
Admin Frontend| HTML / CSS / Vanilla JavaScript
통신| HTTPS / REST / JSON
