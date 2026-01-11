# camstrm
머신비전 카메라 영상 송수신을 구현한 프로젝트입니다.

## 1. 프로젝트 개요

**camstrm**는 Ubuntu 서버에서 여러 대의 산업용 카메라(Hikvision 2대, FLIR 1대)를 **고속으로 grab**하고, “사람이 보기 편한 수준의 fps”로 **Windows 클라이언트에 실시간 송출(cast)**하는 멀티카메라 스트리밍 시스템이다.

핵심 목표:

* 서버에서 카메라 제조사 SDK로 **raw frame을 콜백 기반으로 수신**
* 서버 내부 IPC는 **프로세스 분리(B 방식)**로 안정성을 확보

  * C++ grab 프로세스 ↔ Python WebSocket 브릿지 프로세스를 **UNIX domain socket(UDS)**로 연결
* 클라이언트는 **Qt(C++)**로 WebSocket에 접속해 3개 카메라 영상을 표시
* 스트리밍은 RTSP 대신:

  * 서버 내부: UDS
  * 외부 전송: WebSocket binary
* 네트워크 전송은 **단일 WebSocket 연결(소켓 1개)**로 3대 카메라 프레임을 묶어서 전송(MUX)

---

## 2. 시스템 아키텍처

### 전체 데이터 흐름

```
[Ubuntu Server]
  ├─ (Process A) C++ Camera Grabber
  │     - Hikvision x2 (640x480)
  │     - FLIR x1 (2656x2304)
  │     - SDK callback으로 frame 수신
  │     - 최신 프레임 저장 (per camera)
  │     - 스트리밍용 프레임 번들(FrameBundle) 생성
  │     - UDS로 전송 (/tmp/camstrm.sock)
  │
  └─ (Process B) Python WebSocket Bridge
        - UDS 수신
        - FrameBundle을 WebSocket binary로 브로드캐스트
        - Port: 8765

[Windows Client]
  └─ Qt Viewer (C++)
        - ws://<server-ip>:8765 접속
        - binary FrameBundle 수신
        - 파싱 후 카메라별 이미지 표시 (3개 뷰)
```

---

## 3. 모듈 구성 레이아웃 및 각 역할

### 레포 구조(최종 목표)

```text
camstrm/
├─ CMakeLists.txt
├─ README.md
├─ common/
│  └─ frame_bundle.h
├─ server/
│  ├─ cam_grabber/
│  │  ├─ CMakeLists.txt
│  │  ├─ cam_grabber.cpp          # 메인: 3카메라 최신 프레임 → FrameBundle → UDS 송신
│  │  ├─ camera_device.h          # 공통 인터페이스/Frame 구조체
│  │  ├─ hikvision_camera.h/.cpp  # Hikvision SDK 래퍼(콜백) + open/config/start/stop
│  │  ├─ flir_camera.h/.cpp       # FLIR Spinnaker 래퍼(이벤트 콜백) + open/config/start/stop
│  │  └─ frame_encoder.h/.cpp     # (옵션) raw→JPEG 인코딩 및 downscale 유틸
│  ├─ ws_bridge/
│  │  └─ ws_bridge_server.py      # UDS 수신 → WebSocket broadcast
│  ├─ Dockerfile                  # 서버 컨테이너
│  └─ entrypoint.sh               # 컨테이너에서 ws_bridge + cam_grabber 실행
└─ client/
   └─ qt_viewer/
      ├─ CMakeLists.txt
      ├─ main.cpp                 # 3뷰 UI, viewer 시작
      ├─ camera_cast_viewer.h/.cpp# QWebSocket 수신 + FrameBundle 파싱 + 이미지 업데이트
      └─ widgets/ (옵션)           # 추후 UI 확장 시
```

### 모듈 역할 상세

#### `common/frame_bundle.h`

* 서버/클라이언트 공통 패킷 포맷 정의(바이너리).
* FrameBundleHeader + CameraFrameHeader + JPEG payload들의 연속 구조(MUX).

#### `server/cam_grabber`

* **카메라 SDK 연동(C++)**
* Hikvision 2대 + FLIR 1대 장치 열기, 간단 설정(연속 acquisition, pixel format, ROI 등), 콜백 등록.
* 콜백에서 최신 프레임 버퍼 업데이트.
* 주기적으로(예: 15~30fps) **3카메라 최신 프레임을 묶어 FrameBundle 생성**.
* UDS로 `UdsPacketHeader(size 포함)` + `FrameBundle(payload)` 전송.
* 프로세스는 Python과 분리되어 있어 크래시 격리가 가능해야 함.

#### `server/ws_bridge`

* Python `asyncio` + `websockets`.
* UDS에서 length-prefixed payload를 읽고, payload(FrameBundle)를 그대로 WebSocket binary로 broadcast.
* 다수 클라이언트 연결 허용(브로드캐스트).
* 클라이언트가 없으면 drop 가능(백프레셔 최소화).

#### `client/qt_viewer`

* Qt `QWebSocket`으로 server 접속.
* binary 메시지(FrameBundle) 수신.
* FrameBundle을 파싱해서 카메라별 JPEG 추출 → `QImage::loadFromData()`로 디코드 → UI에 표시.

#### `server/Dockerfile`, `server/entrypoint.sh`

* 서버 전체를 컨테이너로 빌드/실행 가능.
* 컨테이너 시작 시 Python 브릿지 → C++ grabber 순으로 실행.
* 실제 SDK 연동은 환경에 따라 Dockerfile 확장 가능(초기에는 skeleton + dummy 가능).

---

## 4. 세부 요구사항

### 4.1 카메라 구성/입력 요구사항

* 카메라 구성(가정):

  * **Hikvision x2**: 640×480
  * **FLIR x1**: 2656×2304
* 각 카메라는 제조사 SDK를 통해 접근하며, **콜백 기반 프레임 수신 방식**을 사용.

  * Hikvision: MVS SDK (`MvCameraControl.h`) + `MV_CC_RegisterImageCallBackEx`
  * FLIR: Spinnaker SDK + `ImageEventHandler` 기반 이벤트 콜백
* 카메라 serial number 기반으로 특정 장치를 선택해서 open해야 함.

### 4.2 성능/프레임 정책

* 서버(카메라 grab):

  * SDK 수준에서는 고속 grab 가능(예: 300~400fps), 하지만 전체를 네트워크로 보내지는 않음.
* 스트리밍(서버→클라이언트):

  * 사람용 모니터링 목적이므로 **15~30fps 정도면 충분**.
  * 전송 프레임은 **각 카메라별 최신 프레임만 사용**(old frame은 drop).
* 대역폭 절감:

  * 큰 해상도(FLIR 2656×2304)는 스트리밍용으로 downscale 가능(예: 1280×960~1328×1152).
  * 최종 전송은 JPEG 기반을 기본으로 함(클라이언트 디코드 용이).

### 4.3 전송/프로토콜 요구사항

* **외부 네트워크 전송은 WebSocket 1개 연결**로 처리해야 함.
* 한 메시지(패킷) 안에 3개 카메라 프레임을 묶어 전송(MUX).
* WebSocket payload는 FrameBundle 바이너리이며, 내부 포맷은 아래 구조:

```
FrameBundleHeader
  - magic, version, camera_count, seq, timestamp_us
CameraFrameHeader + JPEG bytes (cam0)
CameraFrameHeader + JPEG bytes (cam1)
CameraFrameHeader + JPEG bytes (cam2)
```

* UDS는 스트림이므로 packet framing이 필요:

  * `UdsPacketHeader(magic, size)` + `FrameBundle(size bytes)` 형태로 전송.

### 4.4 안정성/운영 요구사항

* 서버는 **프로세스 분리**로 구성:

  * C++ grabber 크래시/재시작이 Python WebSocket 서버와 분리되어야 함.
* Python 브릿지는 C++가 끊겨도 죽지 않고, “프레임 입력 없음” 상태로 유지 가능.
* (추후) systemd/supervisor로 재시작 정책을 붙이기 쉬운 구조 유지.

### 4.5 빌드/실행 요구사항

* C++ 부분은 CMake 기반.
* Windows 클라이언트는 Qt6(CMake) 기반.
* 서버는 Docker로 빌드/실행 가능:

  * `docker build -t camstrm -f server/Dockerfile .`
  * `docker run --rm -p 8765:8765 camstrm`

### 4.6 구현 범위 단계(권장)

1. **Stage 0 (더미 데이터)**

* C++ grabber가 dummy JPEG payload(문자열/가짜 바이트)를 3카메라 번들로 만들어 송신
* Python 브릿지 + Qt viewer까지 end-to-end 연결 확인

2. **Stage 1 (SDK skeleton 연결)**

* Hikvision/FLIR SDK로 장치 열기 + 콜백 수신
* raw buffer를 최신 프레임으로 저장(전송은 아직 raw 또는 dummy)

3. **Stage 2 (스트리밍 품질)**

* raw → JPEG 인코딩(OpenCV 또는 libjpeg-turbo)
* downscale(특히 FLIR)
* 15~30fps 전송 루프 + “latest only” drop 정책 확정

---

## 5. 명확한 비기능 요구사항(코딩 규칙)

* C++:

  * 최소 C++17.
  * 콜백에서 과도한 작업 금지(최신 프레임 버퍼 업데이트 중심).
  * thread-safety 보장(뮤텍스 또는 lock-free).
* Python:

  * asyncio 기반.
  * 브로드캐스트 시 느린 클라이언트는 drop 정책 적용 가능(최소한 dead socket 정리).
* Qt:

  * UI thread에서 이미지 업데이트.
  * FrameBundle 파싱은 안정적으로(버퍼 길이 검사 필수).

---

이 문서 기준으로 Codex CLI에 “레포 초기화 + 파일 생성 + 스캐폴딩”을 시키면,
바로 **Stage 0 end-to-end 동작**까지 자동으로 뽑아낼 수 있을 거야.

