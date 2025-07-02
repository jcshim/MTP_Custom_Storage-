좋습니다. STM32 USBX 기반 MTP에서 **FileX** 파일 시스템을 사용하는 경우의 **구조와 동작 원리**를 자세히 설명드리겠습니다.

---

## ✅ 기본 구조 개요

USBX의 MTP 클래스는 기본적으로 **ST의 FileX**를 기반으로 구현되어 있습니다.
즉, MTP 명령 처리 중 파일 입출력은 모두 FileX API (`fx_file_open`, `fx_file_read`, `fx_directory_first_entry_find`, 등)를 통해 이루어집니다.

```
             PC (Windows MTP Client)
                     │
            ┌────────▼────────┐
            │ USBX Device Stack│
            └──────┬──────────┘
                   ▼
       ┌──────────────────────┐
       │ ux_device_class_pima │   ← MTP 프로토콜 구현 (PIMA 15740)
       └──────┬───────────────┘
              ▼
        ┌────────────┐
        │   FileX    │   ← 파일 시스템 API (FAT32 지원)
        └────┬───────┘
             ▼
        ┌────────────┐
        │ SD/MMC/eMMC│   ← 실제 저장 장치 (Block I/O)
        └────────────┘
```

---

## ✅ 핵심 작동 원리

### 1. **USBX + PIMA MTP 클래스 초기화**

* STM32CubeIDE에서 USBX 활성화 → **PIMA MTP 클래스 등록**
* `ux_device_stack_initialize()` → `ux_device_class_pima_initialize()` 내부에서 FileX 디렉토리 경로 지정

```c
UX_DEVICE_CLASS_PIMA_PARAMETER pima_parameter;
pima_parameter.ux_device_class_pima_storage[0].ux_device_class_pima_storage_path = "C:/usb_storage/";
```

---

### 2. **MTP 명령 처리 흐름 (예: GetObjectHandles)**

#### ▶️ PC가 보내는 요청: `GetObjectHandles`

1. USBX가 PC로부터 MTP 명령 수신 (`ux_device_stack_interface_control_request`)
2. `ux_device_class_pima_get_object_handles()` 호출됨
3. 이 함수 내부에서 **FileX API로 디렉토리 탐색**

   ```c
   fx_directory_first_entry_find(&media, &entry);
   fx_directory_next_entry_find(&media, &entry);
   ```
4. 결과로 얻은 파일들을 Object Handle 리스트에 추가

#### ✅ 이 단계에서 파일 필터링이 가능

FileX로 얻은 `entry.fx_file_name`의 확장자 분석 후, 조건에 맞는 파일만 Object Handle 목록에 포함

---

### 3. **파일 정보 요청: `GetObjectInfo`**

* 해당 핸들에 대응하는 파일 이름/크기를 **FileX로 조회**
* `fx_file_open`, `fx_file_information_get` 사용
* 응답 구조체 채워서 PC로 전송 (파일명, 크기, 포맷 코드 등)

---

### 4. **파일 데이터 전송: `GetObject`**

* FileX로 해당 파일 열기

  ```c
  fx_file_open(&media, &file, "test.txt", FX_OPEN_FOR_READ);
  ```
* 일정 크기만큼 반복적으로 읽기 (`fx_file_read`)
* USB IN Endpoint로 데이터 전송

---

### 5. **파일 수신: `SendObject`**

* PC → STM32로 파일 보낼 때
* USB로 수신한 데이터 블록을 FileX로 씀 (`fx_file_write`)
* 저장 디렉토리/확장자 조건에 따라 저장 거부/필터링도 가능

---

## ✅ 파일 필터링 구현 포인트 (FileX 기반)

| 위치                                          | 역할       | 필터링 방식                            |
| ------------------------------------------- | -------- | --------------------------------- |
| `ux_device_class_pima_get_object_handles()` | 핸들 목록 생성 | `entry.fx_file_name`으로 확장자 확인     |
| `ux_device_class_pima_get_object_info()`    | 메타정보 전송  | 허용된 파일만 포맷 정보 설정                  |
| `ux_device_class_pima_get_object()`         | 파일 읽기    | 허용되지 않은 파일은 전송 거부                 |
| `ux_device_class_pima_send_object()`        | 파일 쓰기    | `.hwp`, `.txt`, `.doc` 외 저장 거부 가능 |

---

## ✅ 예: 확장자 필터링 (FileX)

```c
if (fx_directory_first_entry_find(&media, &entry) == FX_SUCCESS)
{
    do {
        CHAR* ext = strrchr((char*)entry.fx_file_name, '.');
        if (ext && (
            strcasecmp(ext, ".hwp") == 0 ||
            strcasecmp(ext, ".txt") == 0 ||
            strcasecmp(ext, ".doc") == 0)) {
                
            // 이 파일만 Object Handle에 추가
        }

    } while (fx_directory_next_entry_find(&media, &entry) == FX_SUCCESS);
}
```

---

## ✅ FileX의 주요 장점

| 항목                          | 설명                     |
| --------------------------- | ---------------------- |
| FAT32 완전 지원                 | 롱 파일명, 디렉토리 등 포함       |
| RTOS 최적화                    | ThreadX와 긴밀하게 통합되어 효율적 |
| 낮은 메모리 소비                   | 미니멀 구현 가능              |
| ST 공식 USBX + FileX 통합 예제 다수 | 개발 속도 빠름               |

---

## ✋ 주의 사항

| 항목         | 설명                                                    |
| ---------- | ----------------------------------------------------- |
| 경로 지정      | `"C:/usb_storage"`처럼 루트 경로 반드시 정확히 설정                 |
| 파일 이름 대소문자 | FileX는 기본적으로 **대소문자 구분 안 함**                          |
| 파일 쓰기      | `fx_media_flush()` 또는 `fx_file_close()` 호출해야 실제 저장 완료 |
| 쓰기 충돌 방지   | 동시에 USB와 MCU가 파일을 조작하지 않도록 구조 설계 필요                   |

---

## 📌 결론 요약

| 항목         | 설명                                                                                 |
| ---------- | ---------------------------------------------------------------------------------- |
| 구현 난이도     | 비교적 낮음 (USBX + FileX 연동은 공식 예제 있음)                                                 |
| 확장자 필터링 위치 | `get_object_handles()`에서 가장 쉽고 안정적으로 구현                                            |
| 주요 함수      | `fx_directory_first_entry_find`, `fx_file_open`, `fx_file_read`, `fx_file_write` 등 |

---

필요하시면 아래 중 무엇을 원하시나요?

* ✅ FileX 기반 확장자 필터링이 적용된 실제 코드 예제
* ✅ `ux_device_class_pima_storage_path` 설정 예시
* ✅ STM32CubeIDE 설정 파일 (`.ioc`) 기반 설정 가이드

언제든 요청해 주세요.
