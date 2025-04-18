# SyncRo-API-Tutorial

## Requirements
- SyncRo Package

## RabbitMQ
**SyncRo는 API 통신에 RabbitMQ를 사용합니다.**

SyncRo를 실행하기만 하면, 패키지에 포함된 구성 요소를 통해 RabbitMQ가 자동으로 구동되므로 별도의 설치나 설정은 필요하지 않습니다.

RabbitMQ가 지원하는 언어라면 어떤 언어에서든 자유롭게 연동하여 사용할 수 있습니다.

자세한 사용법은 [rabbitmq.com](rabbitmq.com)을 참고해주세요.

## Initial Setup

### 연결 설정
SyncRo API를 사용하기 위해서는 RabbitMQ 서버에 연결해야 합니다. 기본적으로 SyncRo가 실행되는 컴퓨터의 IP 주소(일반적으로 `localhost`)를 사용합니다.

```python
# RabbitMQ 서버 연결 (기본값은 localhost)
import pika
import json
import uuid
import asyncio

# RabbitMQ 연결 설정
connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost')
)
channel = connection.channel()

# 모듈 이름 설정
my_module_id = "MyModule"

# 응답을 받을 큐 선언
result = channel.queue_declare(queue=my_module_id, exclusive=True)
callback_queue = result.method.queue

# 메시지 수신 콜백 함수
def on_response(ch, method, props, body):
    # 응답 처리 로직
    response = json.loads(body)
    print(f"응답 수신: {response}")

# 메시지 수신 시작
channel.basic_consume(
    queue=callback_queue,
    on_message_callback=on_response,
    auto_ack=True
)

# 비동기 수신을 위한 함수 (실제 사용시 asyncio 이벤트 루프에서 실행)
def start_consuming():
    try:
        channel.start_consuming()
    except KeyboardInterrupt:
        channel.stop_consuming()
```

### 기본 메시지 구조
SyncRo API는 다음과 같은 메시지 형식을 사용합니다:

```python
# 메시지 구조 예시
message = {
    "message_type": "REQUEST",  # "REQUEST", "RESPONSE", "PUBLISH" 중 선택
    "to_module": "RobotModule", # 목적지 모듈 ID
    "from_module": "MyModule",  # 발신 모듈 ID
    "payload": {                # 실제 전송할 데이터
        "request_type": "명령어", # 요청 타입 (connect, movej_by_pose 등)
        # 명령에 필요한 추가 파라미터
    },
    "message_id": "고유ID"      # 메시지 식별을 위한 고유 ID
}
```

### 로봇 모듈 사용 예시

#### 1. 로봇 연결
```python
# 로봇 연결 요청
connect_payload = {
    "request_type": "connect",
    "host": "ROBOT_IP_ADDRESS",  # 로봇 IP 주소
    "robot_maker": "universal_robot"  # 로봇 제조사 (universal_robot, doosan_robot)
}

# 메시지 작성
message = {
    "message_type": "REQUEST",
    "to_module": "RobotModule",
    "from_module": "MyModule",
    "payload": connect_payload,
    "message_id": str(uuid.uuid4())
}

# 메시지 전송
channel.basic_publish(
    exchange='',
    routing_key="RobotModule",  # 목적지 모듈 ID를 라우팅 키로 사용
    properties=pika.BasicProperties(
        reply_to=callback_queue,
        correlation_id=message["message_id"],
    ),
    body=json.dumps(message)
)
```

#### 2. 로봇 움직임 제어
```python
# 로봇 특정 위치로 이동 요청
move_payload = {
    "request_type": "movej_by_pose",
    "target_pose": json.dumps([x, y, z, rx, ry, rz]),  # 목표 위치
    "speed_ratio": 0.5  # 속도 비율 (0.0 ~ 1.0)
}

# 메시지 작성
message = {
    "message_type": "REQUEST",
    "to_module": "RobotModule",
    "from_module": "MyModule",
    "payload": move_payload,
    "message_id": str(uuid.uuid4())
}

# 메시지 전송
channel.basic_publish(
    exchange='',
    routing_key="RobotModule",
    properties=pika.BasicProperties(
        reply_to=callback_queue,
        correlation_id=message["message_id"],
    ),
    body=json.dumps(message)
)
```

#### 3. 로봇 상태 구독
```python
# 로봇 포즈 정보 구독 
subscribe_payload = {
    "request_type": "subscribe",
    "subscibable_value": "pose"  # "pose", "joint", "io", "all" 중 선택
}

# 메시지 작성
message = {
    "message_type": "REQUEST",
    "to_module": "RobotModule",
    "from_module": "MyModule",
    "payload": subscribe_payload,
    "message_id": str(uuid.uuid4())
}

# 메시지 전송
channel.basic_publish(
    exchange='',
    routing_key="RobotModule",
    properties=pika.BasicProperties(
        reply_to=callback_queue,
        correlation_id=message["message_id"],
    ),
    body=json.dumps(message)
)

# 구독 메시지를 지속적으로 수신하려면 start_consuming() 함수를 비동기로 실행
# asyncio.create_task(start_consuming())
```

### 원격 네트워크 설정
로컬 네트워크 외부에서 접속할 경우 공유기의 포트 포워딩 설정이 필요합니다:
1. RabbitMQ 기본 포트(5672 및 15672)를 SyncRo가 실행 중인 컴퓨터의 내부 IP로 포워딩
2. 방화벽 설정에서 해당 포트 개방 확인

## Modules
각 모듈의 이름과 기능은 아래와 같이 구성됩니다.
해당 모듈의 API 명세는 본 문서 뒷부분에서 순차적으로 소개될 예정입니다.

### RobotModule
[링크](https://github.com/portal301/SyncRo-API-Tutorial/blob/main/Modules/RobotModule.md)

## License
This documentation is licensed under the Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International License (CC BY-NC-ND 4.0).

To view a copy of this license, visit https://creativecommons.org/licenses/by-nc-nd/4.0/
