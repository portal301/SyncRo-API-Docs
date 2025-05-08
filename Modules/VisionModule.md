# VisionModule 가이드

## 개요
VisionModule은 SyncRo 시스템에서 카메라 이미지 캡처와 세그멘테이션 처리를 위한 인터페이스를 제공합니다. RabbitMQ 메시징 시스템을 통해 카메라 이미지와 세그멘테이션 데이터를 요청할 수 있습니다.

## 지원하는 기능
- **카메라 이미지 캡처**: depth image 및 rgb 캡처
- **세그멘테이션 처리**: SAM(Segment Anything Model) 기반 이미지 세그멘테이션
---

## API 명령어 요약

| 카테고리 | 명령어 | 필수 매개변수 | 선택 매개변수 | 설명 |
|---------|-------|-------------|------------|------|
| **카메라** | `get_camera_image` | 없음 | 없음 | 깊이 이미지, rgb 캡처 |  
| | `stop_camera_image` | 없음 | 없음 | 이미지 캡쳐 중지 |  
| **세그멘테이션** | `get_segmentation_data` | `points`: 포인트 좌표<br>`labels`: 포인트 레이블 | 없음 | SAM 기반 세그멘테이션 처리 |
| | `stop_segmentation` | 없음 | 없음 | 세그멘테이션 처리 중지 |

---

## 메시지 기본 구조

### 요청 메시지 기본 형식
```python
{
    "message_type": "REQUEST",             # 메시지 타입 (REQUEST/RESPONSE/PUBLISH)
    "to_module": "VisionModule",           # 목적지 모듈
    "from_module": "test_client",             # 발신 모듈
    "payload": {                           # 실제 데이터
        "request_type": "명령어",           # API 명령어
        "data": {} # ... 명령어별 추가 파라미터
    },
    "message_id": "550e8400-e29b-41d4-a716-446655440000"  # 고유 ID
}
```

---

## 기본 사용법

### 1. 카메라 이미지 요청
```python
# 메시지 예시시
message = {
  "from_module": "test_client",
  "to_module": "vision_module",
  "message_type": "REQUEST",
  "payload": {
    "request_type": "get_camera_image",
    "data": {}
  }
}
```
### 3. 세그멘테이션 데이터 요청
```python
message = {
  "from_module": "test_client",
  "to_module": "vision_module",
  "message_type": "REQUEST",
  "payload": {
    "request_type": "stop_camera_image",
    "data": {}
  }
}
```
### 3. 세그멘테이션 데이터 요청
```python
segmentation_payload = {
    "request_type": "get_segmentation_data",
    "data": {
        "points": [[x1, y1], [x2, y2], ...],  # 세그멘테이션 포인트 좌표, 640x480 이미지 기준 가운데 값은 x1 = 320, y1=240. 
        "labels": [1, 1, ...]                  # 포인트 레이블 (1: 전경, 0: 배경)
    }
}
message = {
  "from_module": "test_client",
  "to_module": "vision_module",
  "message_type": "REQUEST",
  "payload": segmentation_payload
}
```

### 4. 세그멘테이션 중지
```python
message = {
  "from_module": "test_client",
  "to_module": "vision_module",
  "message_type": "REQUEST",
  "payload": {
    "request_type": "stop_segmentation",
    "data": {}
  }
}
```

---

## 응답 메시지 구조

### 카메라 이미지 응답
```python
{
    "message_type": "RESPONSE",
    "to_module": "test_client",
    "from_module": "vision_module",
    "payload": {
        "data": {
            "depth_data": base64_encoded_string,    # 깊이 이미지 데이터 
            "intensity_data": base64_encoded_string, # 깊이 이미지 데이터 -> RGB
            "camera_info": {...}, 
            # 카메라 정보 예시: {"cx": 321.233···, "cy": 252.913···, "fx": 524.342···, "fy": 524.346···}
            "shape": [height, width, channels]        # 이미지 크기 예시:[480,640,3]
        },
        "original_sender": "test_client",
        "original_method": "get_camera_image"
    }
}
```

### 세그멘테이션 데이터 응답
```python
{
    "message_type": "RESPONSE",
    "to_module": "test_client",
    "from_module": "vision_module",
    "payload": {
        "data": {
            "depth_data": base64_encoded_string,    # 마스크 적용한 깊이 이미지 데이터 
            "intensity_data": base64_encoded_string, # 깊이 이미지 데이터 -> RGB + contour
            "camera_info": {...}, 
            # 카메라 정보 예시: {"cx": 321.233···, "cy": 252.913···, "fx": 524.342···, "fy": 524.346···}
            "shape": [height, width, channels]        # 이미지 크기 예시:[480,640,3]
        },
        "original_sender": "test_client",
        "original_method": "get_segmentation_data"
    }
}
```

---

## 로그 및 디버깅

VisionModule은 내부적으로 로깅을 수행합니다. 로그는 다음 위치에서 확인할 수 있습니다:

```
logs/vision_module.log
logs/camera_module.log
logs/helios_processor.log
logs/sam_module.log
logs/sam_processor.log
```

로그 형식:
```
YYYY-MM-DD HH:MM:SS - [INFO/ERROR/WARNING] - 메시지
```

### 주요 로그 메시지
- 모듈 시작/종료
- 카메라 이미지 캡처 성공/실패
- 세그멘테이션 처리 시작/완료
- 메시지 처리 오류
- RabbitMQ 연결 상태
