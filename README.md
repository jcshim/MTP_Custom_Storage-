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

