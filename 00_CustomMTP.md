Custom MTP(Media Transfer Protocol) 스택을 STM32 기반 MCU에서 직접 설계하려는 경우, \*\*MTP 프로토콜 명세(PIMA 15740)\*\*를 바탕으로 하여, USB 표준 요청 처리부터 파일 목록 관리, 데이터 송수신, 응답까지 **전체 흐름을 직접 구현**해야 합니다. 아래에 그 구조와 흐름을 **단계별로 시각적 흐름도 개념 + 설명**으로 정리해 드립니다.

---

## ✅ Custom MTP 스택 전체 흐름도 (요약 개념도)

```
    ┌────────────┐
    │   Windows  │
    │(MTP Client)│
    └────┬───────┘
         │ ① USB Setup Packet (MTP Command)
         ▼
┌─────────────────────┐
│ STM32 USB Device RX │
└────────┬────────────┘
         ▼
┌─────────────────────┐
│ MTP Container Parser│ ←─── Header + Payload
└────────┬────────────┘
         ▼
┌──────────────────────────────┐
│ MTP Operation Dispatcher     │  ← 예: GetDeviceInfo, GetObject
└────────┬──────────────┬──────┘
         ▼              ▼
┌─────────────┐   ┌─────────────┐
│ 파일 시스템 │   │ 오브젝트 목록 │
│ (FATFS/SDMMC)│   │ (Object Table)│
└─────────────┘   └─────────────┘
         ▼              ▲
     ⑤ 파일 읽기        │
         ▼              │
┌─────────────────────┐
│ MTP Data Response TX│
└────────┬────────────┘
         ▼
   ⑥ USB IN 전송
         ▼
    ┌────────────┐
    │  Windows   │
    └────────────┘
```

---

## 📌 단계별 설명 (Custom MTP 스택 구성 요소)

### 🔹 ① USB Setup 패킷 수신

* `usbd_mtp.c`에서 `USBD_LL_DataReceived()` 등 콜백으로 **PC가 보낸 요청 패킷**을 수신
* 각 요청은 MTP Container 형식으로 전달됨

  * `OperationCode`, `TransactionID`, `Parameter[]` 등 포함
* 요청 종류 예시:

  * `GetDeviceInfo`, `GetStorageIDs`, `GetObjectHandles`, `GetObject`, `SendObject`, `DeleteObject`

---

### 🔹 ② MTP Container 파싱

* 수신된 `64~512 byte`의 버퍼에서 MTP Container 구조 해석

```c
typedef struct {
  uint32_t length;
  uint16_t type;         // Command, Data, Response 등
  uint16_t opCode;       // ex. 0x1001: GetDeviceInfo
  uint32_t transactionId;
  uint32_t parameters[5];
} MTP_Container_t;
```

* 파싱 후 Dispatcher로 전달

---

### 🔹 ③ MTP Operation Dispatcher

* `opCode` 값을 기준으로 **핸들러 함수** 호출

```c
switch (container.opCode) {
  case MTP_OP_GetDeviceInfo:
    return MTP_Handle_GetDeviceInfo(&container);
  case MTP_OP_GetObjectHandles:
    return MTP_Handle_GetObjectHandles(&container);
  case MTP_OP_GetObject:
    return MTP_Handle_GetObject(&container);
}
```

* Dispatcher는 처리 후 응답(response or data)을 준비

---

### 🔹 ④ 파일 시스템 / 오브젝트 테이블 연동

* `FATFS + SDMMC`로부터 파일 목록 또는 실제 파일을 읽음
* “Object”는 MTP에서 파일/디렉토리를 의미함
* ObjectTable은 다음과 같은 구조로 유지

```c
typedef struct {
  uint32_t handle;
  char filename[64];
  uint32_t size;
  uint16_t formatCode; // ex. 0x3004 = text, 0x3005 = doc
  uint32_t parentHandle;
} MTP_Object_t;
```

* PC 요청에 따라 핸들 목록 또는 메타데이터 전송

---

### 🔹 ⑤ 데이터 전송 준비 (Data Response)

* 요청 유형에 따라:

  * Command 후 `Data Phase` 필요 (예: GetObject)
  * 바로 Response 보내는 경우도 있음
* 데이터를 **USB IN Endpoint로 보내기 위한 버퍼**에 준비

---

### 🔹 ⑥ USB 전송 (IN)

* `USBD_LL_Transmit()` 함수 호출로 MTP 데이터 전송
* 여러 패킷으로 분할 전송 필요 (파일은 수백 KB 이상)

---

## 🧠 설계 시 고려할 핵심 포인트

| 항목                   | 설명                                                                 |
| -------------------- | ------------------------------------------------------------------ |
| **Object Handle 관리** | 파일마다 고유 번호(핸들) 부여 및 조회 필요                                          |
| **MTP 상태 전이 관리**     | Command → Data → Response 순서 보장                                    |
| **파일 필터링**           | `.hwp`, `.doc`, `.txt` 등만 리스트에 포함                                  |
| **대용량 전송**           | 패킷 분할 (bulk endpoint 64/512 byte) 및 offset 기반 전송                   |
| **Windows 호환성**      | `GetDeviceInfo`, `GetObjectHandles`, `GetObjectInfo` 등 주요 요청 구현 필수 |
| **디버깅**              | UART printf, Wireshark USB 분석, Logic Analyzer 병행 사용                |

---

## 🧱 핵심 구현 파일 구조 예시 (Custom 방식)

```
/usb_device
├── usbd_mtp.c / .h         # USB 요청 처리, EndPoint 연결
├── mtp_parser.c / .h       # Container 파싱
├── mtp_dispatcher.c / .h   # 요청별 핸들러 매핑
├── mtp_object.c / .h       # 파일 → Object 변환
├── mtp_fileio.c / .h       # FATFS + 파일 읽기/쓰기
```

---

## 📘 참고 자료 (필수 문서)

| 문서명                                                                                                           | 설명                                     |
| ------------------------------------------------------------------------------------------------------------- | -------------------------------------- |
| [PIMA 15740](https://www.usb.org/document-library/pima-15740-device-independent-protocol-standard)            | MTP 표준 문서                              |
| [USB Device Class Definitions for MTP](https://learn.microsoft.com/en-us/windows-hardware/drivers/usbcon/mtp) | Microsoft MTP 스펙                       |
| STM32 USB Device Library                                                                                      | `usbd_core.c`, `usbd_ctlreq.c` 등 구조 참고 |

---

## ✋ 정리

Custom MTP 스택은 복잡하지만 유연성과 제어력을 극대화할 수 있는 방식입니다.
특히 `.hwp`, `.txt`, `.doc`만 보이게 하거나 특정 폴더만 노출하려면 직접 구현이 적합합니다.

원하신다면 다음도 도와드릴 수 있습니다:

* MTP 명령어별 구현 예시
* 최소 구현 요구 명령 리스트 (`GetDeviceInfo`, `GetObject`, 등)
* 예제 코드 템플릿

진행하고자 하시는 방향(예: MTP 파일 필터링 위주, 최소 동작 MTP 구현 등)을 알려주시면 맞춤 가이드를 드릴 수 있습니다.
