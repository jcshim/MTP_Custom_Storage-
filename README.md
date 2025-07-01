# MTP_Custom_Storage-
MTP_Custom_Storage 

좋습니다! STM32H7B0VBT6 보드에서 **MicroSD 카드에 MTP(Media Transfer Protocol) 방식으로 파일을 저장**하는 과정을 **아주 쉽게** 설명드리겠습니다.

---

## 🔷 목표

> PC에 USB로 연결했을 때
> “디스크 드라이브(F:)”처럼 보이지 않고,
> “휴대폰처럼” 보이면서
> MicroSD에 특정 파일(.txt, .doc, .hwp 등)을 **MTP 방식으로 전송**하는 장치를 만드는 것

---
## https://github.com/STMicroelectronics/stm32-usbx-examples 
---

## 🔷 전체 그림

```plaintext
     [PC]
      │
      │  USB 연결 (MTP 프로토콜)
      ▼
+-----------------------+
| STM32H7B0VBT6 보드    |
|                       |
| ① USB Device (MTP)   ◄──── PC가 인식
| ② 파일 수신          │
| ③ SDMMC로 microSD에 저장
+-----------------------+
```

---

## 🔷 핵심 원리 단계별 설명

### ✅ 1단계: **PC와 통신 – USB MTP 프로토콜**

* **PC는 USB를 통해 STM32를 'MTP 장치'로 인식**합니다.
  휴대폰처럼 보입니다. (예: “Galaxy MTP 장치”처럼)
* 이를 위해 STM32는 **USB Device Stack + MTP Class**를 구현해야 합니다.

> ❗ 기본 USB MSC(Mass Storage)는 "F 드라이브"처럼 보이지만,
> MTP는 "휴대폰 내부 저장소"처럼 보입니다.

---

### ✅ 2단계: **STM32가 파일을 수신**

* PC 사용자가 `.txt`, `.doc` 등의 파일을 복사하면
* STM32는 **MTP 요청을 해석**하고,
* 전송받은 파일 데이터를 RAM(또는 버퍼)에 저장합니다.

> 이 때 `MTP_WriteData_FS()` 같은 함수가 호출됩니다.

---

### ✅ 3단계: **SDMMC로 microSD에 파일 저장**

* STM32는 받은 파일을 **FATFS**를 통해
  **microSD 카드에 저장**합니다.

```c
// 예시 (MTP 수신 후)
f_open(&file, "file.txt", FA_WRITE | FA_CREATE_ALWAYS);
f_write(&file, buffer, length, &bw);
f_close(&file);
```

* 이때 **SDMMC 인터페이스**를 통해 빠르게 저장됩니다.
* SPI보다 빠르고 안정적입니다.

---

### ✅ 4단계: **파일 형식 필터링 (옵션)**

* `.hwp`, `.doc`, `.txt`만 저장되도록
  `filename`을 분석하여 필터링할 수 있습니다.

```c
if (ends_with(filename, ".txt") || ...) {
    // 저장 허용
} else {
    // 무시
}
```

---

## 🧠 요약 한 줄

> **STM32는 USB MTP 장치로 동작하여 PC로부터 파일을 받고,
> 이를 microSD 카드에 SDMMC 인터페이스를 통해 저장하는 장치입니다.**

---

## 💡 보너스: 왜 MTP인가요?

| 방식                                | 설명                | 장점          | 단점        |
| --------------------------------- | ----------------- | ----------- | --------- |
| **MSC (Mass Storage)**            | PC가 SD카드를 직접 엽니다  | 구현 쉬움       | FAT 충돌 위험 |
| **MTP (Media Transfer Protocol)** | STM32가 직접 SD카드 제어 | 안전함, 필터링 가능 | 구현 복잡     |

---

필요하시면:

* STM32CubeIDE 설정 순서
* `usbd_mtp_if.c` 작성법
* FATFS 연동 예제
  도 단계별로 알려드릴 수 있습니다!


STM32H7B0VBT6의 사양에 가장 적합한 MTP 구현 방식은 다음과 같습니다:

---

## ✅ **최적의 구현 방식 요약**

| 항목              | 선택                                        | 이유                                                         |
| --------------- | ----------------------------------------- | ---------------------------------------------------------- |
| **SD 카드 인터페이스** | **SDMMC**                                 | H7B0은 **2× SDMMC** (최대 133 MHz)를 지원하여 **SPI보다 훨씬 빠르고 안정적** |
| **USB 인터페이스**   | **USB Device (USB OTG FS/HS)**            | STM32H7B0은 **USB OTG** 지원 → MTP 구현 가능                      |
| **파일시스템**       | **FATFS**                                 | STM32CubeMX에서 공식 지원, microSD 연동에 표준적                       |
| **MTP 구현 기반**   | **USBX 기반 MTP 예제** (`Ux_Device_PIMA_MTP`) | ST에서 제공, H7 계열에 적합, USB + FATFS 연동 경험 많음                   |
| **OS 사용 여부**    | **RTOS 기반 구현 (USBX)** 권장                  | 복잡한 USB 통신, 파일 I/O 처리에 안정적                                 |

---

## ✅ 추천 구현 리포지토리

> **[STMicroelectronics/stm32-usbx-examples](https://github.com/STMicroelectronics/stm32-usbx-examples)**

* 디렉터리: `/Projects/STM32H7B3I_EVAL/Applications/USBX/Ux_Device_PIMA_MTP`
* 특징:

  * **USBX + ThreadX 기반 MTP 장치 예제**
  * STM32H7 계열용으로 구성
  * **SDMMC + FATFS 연동** 예제로 확장 가능
  * CubeIDE로 열어서 바로 빌드 및 수정 가능

---

## ✅ 구현 흐름

```plaintext
[PC] ⇄ USB MTP ⇄ [STM32H7B0VBT6] ⇄ SDMMC ⇄ microSD
        │                   │
      USBX              FATFS
        │                   │
   usbd_mtp_if.c       f_write()
```

1. USBX로 MTP 요청 수신
2. 요청한 파일명을 `usbd_mtp_if.c`에서 필터링 (`.hwp`, `.txt`, `.doc`)
3. 유효하면 `f_open` + `f_write`로 microSD에 저장

---

## ✅ 장점

* **H7B0의 강력한 성능** 활용 (Cortex-M7 + L1 cache + DMA)
* **고속 USB + SDMMC** 조합 → 빠르고 안정적
* ST에서 제공하는 예제를 사용하면 **호환성 확보 및 유지보수 용이**

---

## ✅ 결론

🔷 STM32H7B0VBT6에 가장 적합한 방법은
**ST의 USBX 기반 MTP 예제를 사용하여, SDMMC 인터페이스로 microSD에 FATFS 방식으로 저장하는 구조**입니다.

> 필요 시: 위 예제를 H7B0VBT6 보드에 맞게 **포팅 방법**과
> `usbd_mtp_if.c` 코드 샘플도 단계별로 알려드릴 수 있습니다.

좋은 질문입니다. \*\*`USBX`\*\*와 \*\*`Ux_Device_PIMA_MTP`\*\*는 관련되어 있지만 서로 다른 범위와 역할을 가지고 있습니다. 차이를 아래와 같이 명확하게 정리할 수 있습니다:

---

## ✅ USBX vs. Ux\_Device\_PIMA\_MTP 차이점

| 항목               | **USBX**                                              | **Ux\_Device\_PIMA\_MTP**                                            |
| ---------------- | ----------------------------------------------------- | -------------------------------------------------------------------- |
| **정의**           | USB 장치/호스트를 위한 **전체 USB 스택(미들웨어 프레임워크)**              | USBX 위에서 동작하는 **MTP(미디어 전송 프로토콜) 클래스 구현 예제**                         |
| **역할**           | USB 통신의 기본 구조와 동작 관리 (클래스, 파이프, 엔드포인트, 요청 처리 등)       | USBX에 등록된 하나의 \*\*장치 클래스(Device Class)\*\*로, PC와 MTP 방식으로 파일 주고받기 처리 |
| **제공 범위**        | 모든 USB 장치 클래스(CDC, HID, MSC, Audio, MTP 등)와 관련된 기반 코드 | MTP(PIMA 15740) 표준을 구현한 클래스 코드 및 예제                                  |
| **기반 위치**        | STM32CubeIDE에서 `USBX Core`로 설정                        | `USBX Device Classes → PIMA_MTP`로 설정                                 |
| **STCube 설정 명칭** | `USBX` 또는 `ThreadX/USBX`                              | `Ux_Device_PIMA_MTP`                                                 |
| **종속성**          | 자체적으로는 클래스가 아님 → 클래스를 포함하거나 조율함                       | **USBX에 의존**하며, 자체적으로는 독립 실행 불가                                      |
| **실제 기능**        | USB 장치/호스트 초기화, 이벤트 처리, 스케줄링, 메모리 관리 등                | PC가 전송하는 파일 요청/업로드 명령을 해석하고 내부 저장소(SD카드 등)와 연동함                      |

---

## ✅ 비유로 설명하면:

* `USBX`는 **USB 시스템 전체를 위한 운영체제 내장형 드라이버 프레임워크**입니다. (즉, "자동차의 뼈대")
* `Ux_Device_PIMA_MTP`는 \*\*그 위에 올려진 미디어 전송 기능을 수행하는 하나의 애플리케이션(클래스)\*\*입니다. (즉, "운전 기능 중 라디오")

---

## ✅ 구조 상 관계

```
+---------------------+
|     USBX Core       |  ← USB 장치 드라이버 스택
|  (ThreadX 기반)     |
+----------+----------+
           |
           v
+---------------------+
| Ux_Device_PIMA_MTP  |  ← MTP 장치 클래스 (사진/문서 전송용)
+----------+----------+
           |
           v
+---------------------+
|  User Storage Logic |  ← FATFS, Flash, RAM 등 파일 시스템 연동
+---------------------+
```

* USBX가 **기반 USB 장치 스택**을 구성하고
* MTP 클래스(Ux\_Device\_PIMA\_MTP)는 그 위에 올라가서 **파일 전송 요청**을 처리하며
* 사용자 정의 스토리지 연동 코드에서 **microSD 등과 FATFS를 사용**합니다

---

## ✅ 실제 개발 시 체크포인트

| 체크 항목                    | 설명                                                          |
| ------------------------ | ----------------------------------------------------------- |
| CubeIDE에서 USBX 설정        | Core + MTP 클래스(PIMA) 활성화 필요                                 |
| `ux_device_class_pima.c` | MTP 클래스 동작 처리                                               |
| `ux_device_stack.c`      | USBX 핵심 장치 프레임워크 코드                                         |
| 사용자 저장소 구현               | `ux_device_class_pima_storage.c` 또는 유사한 위치에서 FATFS 연동 구현 필요 |

---

## ✅ 결론 요약

| 용어                        | 정의                                      |
| ------------------------- | --------------------------------------- |
| **USBX**                  | USB 통신 전체를 처리하는 미들웨어 프레임워크 (ThreadX 기반) |
| **Ux\_Device\_PIMA\_MTP** | USBX 위에서 실행되는 MTP 장치 클래스 구현 및 예제        |

즉, `USBX`는 "플랫폼"이고, `Ux_Device_PIMA_MTP`는 "MTP 앱(모듈)"입니다.

---

필요하시면 `ux_device_class_pima_storage` 내에서 **FATFS와 연동하는 커스터마이징 방법**도 함께 설명드릴 수 있습니다.

