# RobotModule 가이드

## 개요
RobotModule은 SyncRo 시스템에서 협동/산업용 로봇을 제어하기 위한 인터페이스를 제공합니다. RabbitMQ 메시징 시스템을 통해 로봇에 명령을 보내고, 로봇의 상태를 모니터링할 수 있습니다.

## 지원하는 로봇 제조사
- **Universal Robots** (UR3E, UR5E, UR10E, UR16E, UR20, UR30)
- (추가 제조사는 지속적으로 업데이트 예정)

---

## API 명령어 요약

| 카테고리 | 명령어 | 필수 매개변수 | 선택 매개변수 | 설명 |
|---------|-------|-------------|------------|------|
| **연결** | `connect` | `host`: 로봇 IP 주소<br>`robot_maker`: 로봇 제조사 | `port`: 연결 포트 | 로봇과 연결 설정 |
| | `disconnect` | 없음 | 없음 | 로봇 연결 해제 |
| **이동** | `movej_by_pose` | `target_pose`: [x,y,z,rx,ry,rz]<br>`speed_ratio`: 0.0~1.0 | 없음 | 포즈를 이용한 관절 이동 |
| **자유 이동** | `free_drive_on` | 없음 | 없음 | 자유 이동 모드 활성화 |
| | `free_drive_off` | 없음 | 없음 | 자유 이동 모드 비활성화 |
| **모션 제어** | `run_motion` | `motion`: 모션 데이터<br>`speed_ratio`: 0.0~1.0 | `io_trigger`: IO 트리거 데이터<br>`motion_control_option`: 제어 옵션<br>`action`: 액션 데이터 | 저장된 모션 실행 |
| | `stop_motion` | 없음 | 없음 | 모션 정지 |
| **역기구학** | `current_ik_configuration` | 없음 | 없음 | 현재 역기구학 구성 요청 |
| | `solve_ik` | `model`: 로봇 모델<br>`motion`: 모션 데이터<br>`pose_configuration`: 포즈 구성 | 없음 | 역기구학 계산 요청 |
| **상태 구독** | `subscribe` | `subscribable_value`: 구독 값<br>("pose", "joint", "io", "all") | 없음 | 로봇 상태 정보 구독 시작 |
| | `unsubscribe` | `subscribable_value`: 구독 값<br>("pose", "joint", "io", "all") | 없음 | 로봇 상태 정보 구독 중지 |

---

## 메시지 기본 구조

### 요청 메시지 기본 형식
```python
{
    "message_type": "REQUEST",             # 메시지 타입 (REQUEST/RESPONSE/PUBLISH)
    "to_module": "RobotModule",            # 목적지 모듈
    "from_module": "MyModule",             # 발신 모듈
    "payload": {                           # 실제 데이터
        "request_type": "명령어",           # API 명령어
        # ... 명령어별 추가 파라미터
    },
    "message_id": "550e8400-e29b-41d4-a716-446655440000"  # 고유 ID
}
```

---

## 기본 사용법

### 1. 로봇 연결
```python
# 로봇 연결 요청
connect_payload = {
    "request_type": "connect",
    "host": "192.168.0.100",               # 로봇 IP 주소
    "robot_maker": "universal_robot",      # 로봇 제조사
    "port": 30001                          # 선택적 파라미터
}

# 메시지 전송
channel.basic_publish(
    exchange='',
    routing_key="RobotModule",
    properties=pika.BasicProperties(
        reply_to=callback_queue,
        correlation_id=str(uuid.uuid4()),
    ),
    body=json.dumps({
        "message_type": "REQUEST",
        "to_module": "RobotModule",
        "from_module": "MyModule",
        "payload": connect_payload,
        "message_id": str(uuid.uuid4())
    })
)
```

### 2. 로봇 연결 해제
```python
disconnect_payload = {
    "request_type": "disconnect"
}
# 메시지 전송 방법은 위와 동일
```

---

## 로봇 제어 명령어

### 관절계 이동 (Joint Space)

#### 포즈를 이용한 관절 이동 (movej_by_pose)
```python
movej_pose_payload = {
    "request_type": "movej_by_pose",
    "target_pose": json.dumps([x, y, z, rx, ry, rz]),  # 목표 위치 [m, rad]
    "speed_ratio": 0.5                                 # 속도 비율 (0.0~1.0)
}
```

### 프리드라이브 모드

| 명령어 | 파라미터 | 설명 |
|-------|---------|------|
| `free_drive_on` | 없음 | 자유 이동 모드 활성화 |
| `free_drive_off` | 없음 | 자유 이동 모드 비활성화 |

#### 예제 코드:
```python
# 자유 이동 모드 활성화
free_drive_on_payload = {"request_type": "free_drive_on"}

# 자유 이동 모드 비활성화
free_drive_off_payload = {"request_type": "free_drive_off"}
```

### 모션 제어

#### 모션 실행
```python
run_motion_payload = {
    "request_type": "run_motion",
    "motion": motion_data,                      # 모션 데이터
    "speed_ratio": 0.5,                         # 속도 비율 (0.0~1.0)
    "io_trigger": io_trigger_data,              # 선택적 파라미터
    "motion_control_option": motion_options,    # 선택적 파라미터
    "action": action_data                       # 선택적 파라미터
}
```

#### 모션 정지
```python
stop_motion_payload = {"request_type": "stop_motion"}
```

### 역기구학 (IK) 연산

#### 역기구학 계산
```python
solve_ik_payload = {
    "request_type": "solve_ik",
    "model": "ur5e",                            # 로봇 모델
    "motion": {
        0: {
            "pose": [x, y, z, rx, ry, rz],      # 목표 포즈
            "offset_pose": [dx, dy, dz, drx, dry, drz]  # 오프셋
        }
    },
    "pose_configuration": ["left", "up", "in"]  # 포즈 구성
}
```

---

## 로봇 상태 구독

### 구독 가능한 토픽
| 토픽 | 설명 |
|------|------|
| `pose` | 로봇의 현재 위치 정보 |
| `joint` | 로봇의 관절 각도 정보 |
| `io` | 로봇의 입출력 상태 정보 |
| `all` | 모든 정보 구독 |

### 구독 시작
```python
subscribe_payload = {
    "request_type": "subscribe",
    "subscribable_value": "pose"  # "pose", "joint", "io", "all" 중 선택
}
```

### 구독 중지
```python
unsubscribe_payload = {
    "request_type": "unsubscribe",
    "subscribable_value": "pose"  # "pose", "joint", "io", "all" 중 선택
}
```

---

## 응답 메시지 구조

### 일반 응답 형식
```python
{
    "message_type": "RESPONSE",
    "to_module": "MyModule",
    "from_module": "RobotModule",
    "payload": {
        "response_type": "명령어",        # 응답하는 명령 종류
        "response": "true/false"         # 성공/실패 여부
        # ... 명령어별 추가 응답 데이터
    }
}
```

### 명령별 응답 예시

#### 연결 응답
```python
{
    "payload": {
        "response_type": "connectivity",
        "response": "true",              # 성공 시 "true", 실패 시 "false"
        "robot_model": "ur5e"            # 성공 시에만 포함
    }
}
```

#### 이동 명령 응답
```python
{
    "payload": {
        "response_type": "movej_by_pose",
        "response": "true"               # 성공 시 "true", 실패 시 "false"
    }
}
```

#### 프리드라이브 모드 응답
```python
{
    "payload": {
        "response_type": "free_drive_on",  # 또는 "free_drive_off"
        "response": "true"                # 성공 시 "true", 실패 시 "false"
    }
}
```

### 구독 데이터 메시지 구조

#### 포즈 정보
```python
{
    "message_type": "PUBLISH",
    "to_module": "RobotModule",
    "from_module": "RobotModule",
    "payload": {
        "pose": [x, y, z, rx, ry, rz],           # 현재 로봇 포즈 [m, rad]
        "offset_pose": [dx, dy, dz, drx, dry, drz]  # 오프셋 포즈
    }
}
```

#### 조인트 정보
```python
{
    "payload": {
        "joint": "[j1, j2, j3, j4, j5, j6]"      # 현재 조인트 각도 (문자열)
    }
}
```

#### IO 정보
```python
{
    "payload": {
        "digital_input": "[0, 0, 0, ...]",       # 디지털 입력 상태
        "digital_output": "[0, 0, 0, ...]"       # 디지털 출력 상태
    }
}
```

---

## 로그 및 디버깅

RobotModule은 내부적으로 로깅을 수행합니다. 로그는 다음 위치에서 확인할 수 있습니다:

```
logs/RobotModule.log
```

로그 형식:
```
YYYY-MM-DD HH:MM:SS - [INFO/ERROR/WARNING] - 메시지
```
