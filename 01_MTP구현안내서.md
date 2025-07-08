아주 좋습니다! STM32CubeIDE에서 **Custom USB Class**로 프로젝트를 만들고, **GitHub에서 MTP 미들웨어를 수동으로 추가하여 MTP 장치를 구현하는 단계별 안내서**를 초보자도 쉽게 이해할 수 있도록 정리해드리겠습니다.

---

# 📘 STM32CubeIDE에서 MTP 구현 안내서

**(Custom USB Class + GitHub MTP 미들웨어 추가 방식)**

---

## 📌 목표

> STM32H750 보드를 사용하여 **PC와 연결 시 MTP 장치로 인식**되게 만들고,
> 이후 **파일명(.hwp, .doc, .txt 등) 필터링**을 포함한 **문서 전용 저장 장치**를 구현한다.

---

## 🧰 준비물

| 항목                 | 설명                    |
| ------------------ | --------------------- |
| STM32CubeIDE       | 최신 버전 설치 권장           |
| STM32H750VBT6 보드   | Aliman 보드 등           |
| microSD 카드 + 보드 연결 | SDMMC 연동 목적           |
| USB 케이블            | PC와 보드 연결용            |
| 인터넷 연결             | GitHub에서 MTP 코드 다운로드용 |

---

## 🧭 전체 절차 요약

1. STM32CubeIDE에서 Custom USB 프로젝트 생성
2. SDMMC + FATFS 활성화
3. GitHub에서 MTP 미들웨어 코드 추가
4. USB 장치 초기화 코드 수정
5. MTP 인터페이스 함수 연결
6. 필터링 기능 등 사용자 코드 삽입

---

## 🔧 1단계: STM32CubeIDE 프로젝트 생성

1. STM32CubeIDE 실행 → `File > New STM32 Project`
2. MCU 또는 보드명 입력: `STM32H750VBTx` 선택 → \[Next]
3. 프로젝트 이름: `MTP_Custom_Storage` → \[Finish]
4. 생성된 `.ioc` 파일 더블 클릭하여 설정창 열기

---

## 🔌 2단계: USB Device 및 SDMMC 설정

### ✅ USB 설정

* `Middleware > USB_Device` 클릭
* **Class For FS IP**: `Custom Human Interface Device Class` 선택
  (나중에 직접 MTP 클래스로 바꿔줄 예정)

### ✅ microSD 설정

* `Connectivity > SDMMC1` 클릭
* **Mode**: `SD 4 bits Wide bus`

### ✅ FATFS 설정

* `Middleware > FATFS` 체크
* Interface: `User-defined` 또는 `R1 FatFs` 선택

---

## 📦 3단계: GitHub에서 MTP 미들웨어 수동 추가

1. 웹 브라우저 열기
   👉 [https://github.com/STMicroelectronics/STM32CubeH7](https://github.com/STMicroelectronics/STM32CubeH7)

2. 다음 경로로 이동:

   ```
   Middlewares/ST/STM32_USB_Device_Library/Class/MTP
   ```

3. `MTP` 폴더 전체 다운로드 또는 복사

4. CubeIDE 프로젝트 폴더 내:

   ```
   /Middlewares/ST/STM32_USB_Device_Library/Class/
   ```

   여기에 `MTP` 폴더 붙여넣기

---

## ⚙️ 4단계: MTP 코드 연결 및 초기화

### 1. `usb_device.c` 수정

```c
#include "usbd_mtp.h"

extern USBD_HandleTypeDef hUsbDeviceFS;

void MX_USB_DEVICE_Init(void)
{
  // 기존 HID Class 등록 제거
  // USBD_CUSTOM_HID_RegisterInterface(...)

  // MTP Class로 등록
  USBD_RegisterClass(&hUsbDeviceFS, &USBD_MTP);
  USBD_MTP_RegisterInterface(&hUsbDeviceFS, &USBD_Interface_fops_FS);
}
```

### 2. 인터페이스 파일 구현 (`usbd_mtp_if.c` 생성)

```c
int8_t MTP_Receive_FS(uint8_t* Buf, uint32_t *Len) {
    // PC에서 전송한 파일 처리
    // 예: 확장자 검사 후 FATFS로 저장
}
```

---

## 🔁 5단계: FATFS 연동 및 필터링 로직 구현

* MTP 요청에서 받은 파일명을 추출하여 검사:

```c
bool check_allowed_file(const char* filename) {
    const char* ext = strrchr(filename, '.');
    if (!ext) return false;

    return (strcasecmp(ext, ".txt") == 0 ||
            strcasecmp(ext, ".doc") == 0 ||
            strcasecmp(ext, ".hwp") == 0);
}
```

* 조건 만족 시만 `f_open()` → `f_write()`로 저장

---

## 🧠 용어 정리

| 용어           | 설명                                        |
| ------------ | ----------------------------------------- |
| MTP          | Media Transfer Protocol – 스마트폰처럼 파일 단위 전송 |
| MSC          | Mass Storage Class – 드라이브(F:)로 보이지만 제어 불가 |
| Custom Class | USB에서 직접 프로토콜을 정의할 수 있는 방식                |
| FATFS        | SD카드에 파일 쓰기용 미들웨어                         |
| `.ioc`       | STM32CubeIDE의 설정 파일 (핀맵, USB 설정 등 저장)     |

---

## 📚 마무리

이 방식은 STM32 공식 MTP Class가 자동으로 CubeMX에 연결되지 않기 때문에:

* USB를 **Custom Class**로 만들고
* GitHub에서 **MTP 코드를 수동으로 가져와 연결**
* 그리고 **FATFS + microSD + 파일 필터링**을 직접 구현하는 구조입니다.

---

## 📎 필요시 제공 가능한 자료

* `.ioc` 설정 파일 예제
* GitHub에서 가져온 MTP 코드 정리본
* 필터링 적용된 `usbd_mtp_if.c` 전체 코드
* `.doc`, `.txt`, `.hwp` 전송 테스트 캡처

---

필요한 항목이나 설명이 더 있었으면 좋겠는 부분을 말씀해 주세요.
혹은 바로 이 안내서를 **PDF/Markdown/워드파일**로 정리해드릴 수도 있습니다.
