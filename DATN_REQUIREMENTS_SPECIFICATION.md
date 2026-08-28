# ĐẶC TẢ THIẾT KẾ CHI TIẾT (LOW-LEVEL DESIGN SPEC)

## Thiết kế và triển khai hệ thống giám sát cảm biến và chẩn đoán lỗi trên ô tô theo tiêu chuẩn AUTOSAR Classic

**Mục đích:** File này liệt kê chi tiết đến mức **từng file, từng include, từng struct/enum, từng hàm cần viết** (tên hàm, prototype, input/output, mô tả), để mỗi thành viên đọc vào là biết chính xác cần code những gì, không cần suy diễn thêm. Cấu trúc thư mục tương ứng với mục 25 của đề cương dự án.

**Quy ước đặt tên:** `<ModuleName>_<Action>()`, ví dụ `Dem_SetEventStatus()`, `IoHwAb_ReadCoolantTemp()`. Kiểu trả về lỗi dùng chung `Std_ReturnType` (E_OK = 0, E_NOT_OK = 1) mô phỏng theo quy ước AUTOSAR.

---

## 0. Kiểu dữ liệu dùng chung (Common Types)

### File: `Common/Std_Types.h`

```c
#ifndef STD_TYPES_H
#define STD_TYPES_H

#include <stdint.h>
#include <stdbool.h>

typedef uint8_t Std_ReturnType;
#define E_OK      0x00u
#define E_NOT_OK  0x01u

typedef enum {
    SIGNAL_QUALITY_VALID = 0,
    SIGNAL_QUALITY_INVALID,
    SIGNAL_QUALITY_TIMEOUT
} SignalQuality_t;

#endif /* STD_TYPES_H */
```

---

## 1. `Common/` — Định nghĩa dùng chung giữa các node

### 1.1. File: `Common/CanIds.h`

**Include:** `<stdint.h>`

**Nội dung:**

```c
#ifndef CAN_IDS_H
#define CAN_IDS_H

/* CAN ID definitions */
#define CANID_SENSOR_DATA          0x100u   /* ECU -> GUI, cycle 100ms  */
#define CANID_DIAGNOSTIC_STATUS    0x200u   /* ECU -> GUI, event + 1000ms heartbeat */
#define CANID_ECU_STATUS           0x300u   /* ECU -> GUI, cycle 500ms */
#define CANID_FAULT_INJECTION_CMD  0x400u   /* GUI -> ECU, event */
#define CANID_OBD_REQUEST          0x500u   /* OBD Reader -> ECU, on-demand */
#define CANID_OBD_RESPONSE         0x501u   /* ECU -> OBD Reader, on-demand */

/* DLC definitions */
#define DLC_SENSOR_DATA             8u
#define DLC_DIAGNOSTIC_STATUS       8u
#define DLC_ECU_STATUS              4u
#define DLC_FAULT_INJECTION_CMD     2u
#define DLC_OBD_REQUEST             3u
#define DLC_OBD_RESPONSE            8u

/* Cycle times (ms) */
#define CYCLE_SENSOR_DATA_MS        100u
#define CYCLE_DIAGNOSTIC_HEARTBEAT_MS 1000u
#define CYCLE_ECU_STATUS_MS         500u

#endif /* CAN_IDS_H */
```

### 1.2. File: `Common/DtcList.h`

**Include:** `<stdint.h>`

**Nội dung:**

```c
#ifndef DTC_LIST_H
#define DTC_LIST_H

typedef enum {
    DTC_ID_P0217_COOLANT_OVERTEMP = 0,
    DTC_ID_P0522_OIL_PRESSURE_LOW,
    DTC_ID_P0562_BATTERY_UNDERVOLT,
    DTC_ID_P0116_COOLANT_INVALID,
    DTC_ID_P0117_SENSOR_TIMEOUT,
    DTC_ID_U0100_CAN_TIMEOUT,
    DTC_ID_COUNT                     /* luôn để cuối, dùng để lặp mảng */
} DtcId_t;

typedef struct {
    DtcId_t     id;
    uint16_t    code;      /* mã dạng số, ví dụ 0x0217 */
    const char *shortName; /* dùng để in ra GUI/OBD-II Reader */
} DtcInfo_t;

/* Bảng tra cứu tĩnh (định nghĩa trong DtcList.c hoặc dùng static const trong .h) */
extern const DtcInfo_t Dtc_Table[DTC_ID_COUNT];

#endif /* DTC_LIST_H */
```

### 1.3. File: `Common/DtcList.c`

```c
#include "DtcList.h"

const DtcInfo_t Dtc_Table[DTC_ID_COUNT] = {
    { DTC_ID_P0217_COOLANT_OVERTEMP, 0x0217, "COOLANT OVER TEMP" },
    { DTC_ID_P0522_OIL_PRESSURE_LOW, 0x0522, "OIL PRESSURE LOW" },
    { DTC_ID_P0562_BATTERY_UNDERVOLT,0x0562, "BATTERY UNDER VOLT" },
    { DTC_ID_P0116_COOLANT_INVALID,  0x0116, "COOLANT SIGNAL INVALID" },
    { DTC_ID_P0117_SENSOR_TIMEOUT,   0x0117, "SENSOR TIMEOUT" },
    { DTC_ID_U0100_CAN_TIMEOUT,      0x0100, "CAN COMM TIMEOUT" },
};
```

### 1.4. File: `Common/SignalDefs.h`

**Include:** `<stdint.h>`

**Nội dung:**

```c
#ifndef SIGNAL_DEFS_H
#define SIGNAL_DEFS_H

/* Scale factors dùng khi đóng gói/giải mã signal vào CAN payload */
#define COOLANT_TEMP_SCALE       10   /* °C * 10, signed int16 */
#define OIL_PRESSURE_SCALE       10   /* kPa * 10, uint16 */
#define BATTERY_VOLTAGE_SCALE    100  /* V * 100, uint16 */

/* Physical thresholds (dùng chung để GUI/ECU không lệch số liệu khi hiển thị debug) */
#define COOLANT_TEMP_MAX_C       110
#define OIL_PRESSURE_MIN_KPA     150
#define BATTERY_VOLTAGE_MIN_V    11.0f

/* Bit mask cho byte 6 của Sensor Data Message (0x100) */
#define SENSOR_FLAG_COOLANT_VALID   (1u << 0)
#define SENSOR_FLAG_OIL_VALID       (1u << 1)
#define SENSOR_FLAG_BATTERY_VALID   (1u << 2)

#endif /* SIGNAL_DEFS_H */
```

---

## 2. `ECU_Firmware/Mcal/` — Microcontroller Abstraction Layer

### 2.1. File: `Mcal/Adc/Adc.h` / `Adc.c`

**Include:** `"Std_Types.h"`, HAL vendor (`stm32f1xx_hal.h`)

**Enum kênh ADC:**

```c
typedef enum {
    ADC_CH_COOLANT = 0,
    ADC_CH_OIL_PRESSURE,
    ADC_CH_BATTERY_VOLTAGE,
    ADC_CH_COUNT
} AdcChannel_t;
```

**Hàm cần triển khai:**

| Hàm                | Prototype                                                                 | Mô tả                                                                            |
| ------------------- | ------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| Adc_Init            | `Std_ReturnType Adc_Init(void);`                                        | Khởi tạo ADC peripheral, cấu hình các kênh                                   |
| Adc_ReadChannel     | `Std_ReturnType Adc_ReadChannel(AdcChannel_t ch, uint16_t *rawValue);`  | Đọc giá trị ADC thô (0–4095 với 12-bit) của 1 kênh, trả về qua con trỏ |
| Adc_ReadAllChannels | `Std_ReturnType Adc_ReadAllChannels(uint16_t rawValues[ADC_CH_COUNT]);` | Đọc toàn bộ kênh trong 1 lần gọi (dùng trong main loop định kỳ)         |

### 2.2. File: `Mcal/Dio/Dio.h` / `Dio.c` (tùy chọn, nếu dùng Door Status trong tương lai)

| Hàm            | Prototype                                                            | Mô tả                       |
| --------------- | -------------------------------------------------------------------- | ----------------------------- |
| Dio_Init        | `Std_ReturnType Dio_Init(void);`                                   | Khởi tạo GPIO input cho DIO |
| Dio_ReadChannel | `Std_ReturnType Dio_ReadChannel(uint8_t channel, uint8_t *value);` | Đọc mức digital 0/1        |

### 2.3. File: `Mcal/Can/Can.h` / `Can.c`

**Include:** `"Std_Types.h"`, `stm32f1xx_hal_can.h`

**Struct frame CAN dùng nội bộ:**

```c
typedef struct {
    uint32_t id;
    uint8_t  dlc;
    uint8_t  data[8];
} CanFrame_t;

typedef void (*Can_RxCallback_t)(const CanFrame_t *frame);
```

**Hàm cần triển khai:**

| Hàm                                         | Prototype                                                            | Mô tả                                                                                                    |
| -------------------------------------------- | -------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| Can_Init                                     | `Std_ReturnType Can_Init(void);`                                   | Khởi tạo bxCAN peripheral (baudrate 500kbps), cấu hình filter nhận tất cả ID cần thiết            |
| Can_Transmit                                 | `Std_ReturnType Can_Transmit(const CanFrame_t *frame);`            | Gửi 1 frame ra bus CAN (dùng HAL_CAN_AddTxMessage bên dưới)                                           |
| Can_RegisterRxCallback                       | `void Can_RegisterRxCallback(Can_RxCallback_t cb);`                | Đăng ký hàm callback được gọi khi nhận frame mới (từ ISR hoặc polling)                         |
| Can_MainFunction                             | `void Can_MainFunction(void);`                                     | Gọi định kỳ trong main loop để xử lý polling nhận frame (nếu không dùng interrupt hoàn toàn) |
| (Internal) HAL_CAN_RxFifo0MsgPendingCallback | `void HAL_CAN_RxFifo0MsgPendingCallback(CAN_HandleTypeDef *hcan);` | Callback HAL chuẩn của STM32, bên trong gọi`Can_RxCallback_t` đã đăng ký                        |

### 2.4. File: `Mcal/Eeprom/Eeprom_I2C.h` / `Eeprom_I2C.c`

**Include:** `"Std_Types.h"`, `stm32f1xx_hal_i2c.h`

| Hàm              | Prototype                                                                               | Mô tả                                         |
| ----------------- | --------------------------------------------------------------------------------------- | ----------------------------------------------- |
| Eeprom_Init       | `Std_ReturnType Eeprom_Init(void);`                                                   | Khởi tạo I2C, kiểm tra kết nối AT24C32     |
| Eeprom_WriteBytes | `Std_ReturnType Eeprom_WriteBytes(uint16_t addr, const uint8_t *data, uint16_t len);` | Ghi mảng byte vào EEPROM tại địa chỉ addr |
| Eeprom_ReadBytes  | `Std_ReturnType Eeprom_ReadBytes(uint16_t addr, uint8_t *data, uint16_t len);`        | Đọc mảng byte từ EEPROM                     |

---

## 3. `ECU_Firmware/EcuAbstraction/` — ECU Abstraction Layer

### 3.1. File: `EcuAbstraction/IoHwAb/IoHwAb.h` / `IoHwAb.c`

**Include:** `"Std_Types.h"`, `"Adc.h"`

**Struct kết quả đọc cảm biến:**

```c
typedef struct {
    float           coolantTempC;
    float           oilPressureKpa;
    float           batteryVoltageV;
    SignalQuality_t coolantQuality;
    SignalQuality_t oilQuality;
    SignalQuality_t batteryQuality;
} IoHwAb_SensorData_t;
```

**Hàm cần triển khai:**

| Hàm                        | Prototype                                                      | Mô tả                                                                                                             |
| --------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| IoHwAb_Init                 | `Std_ReturnType IoHwAb_Init(void);`                          | Gọi Adc_Init(), khởi tạo trạng thái nội bộ                                                                   |
| IoHwAb_ReadCoolantTemp      | `Std_ReturnType IoHwAb_ReadCoolantTemp(float *tempC);`       | Đọc ADC kênh Coolant, áp dụng công thức quy đổi (tuyến tính hoặc bảng tra NTC) sang °C                |
| IoHwAb_ReadOilPressure      | `Std_ReturnType IoHwAb_ReadOilPressure(float *pressureKpa);` | Đọc ADC kênh Oil, quy đổi sang kPa                                                                             |
| IoHwAb_ReadBatteryVoltage   | `Std_ReturnType IoHwAb_ReadBatteryVoltage(float *voltageV);` | Đọc ADC kênh Battery, quy đổi theo tỉ lệ mạch chia áp                                                      |
| IoHwAb_ReadAll              | `Std_ReturnType IoHwAb_ReadAll(IoHwAb_SensorData_t *data);`  | Gọi 3 hàm trên, gán luôn giá trị Quality dựa trên dải hợp lệ ADC thô (0V/VCC bất thường → INVALID) |
| IoHwAb_RawToTemp (internal) | `static float IoHwAb_RawToTemp(uint16_t raw);`               | Hàm quy đổi nội bộ (tuyến tính hoặc Steinhart-Hart nếu dùng NTC thật)                                    |

### 3.2. File: `EcuAbstraction/CanIf/CanIf.h` / `CanIf.c` + `CanIf_Cfg.h`

**Include:** `"Std_Types.h"`, `"Can.h"`, `"CanIds.h"`

**File `CanIf_Cfg.h` — bảng ánh xạ PDU ↔ CAN ID:**

```c
typedef enum {
    PDU_ID_SENSOR_DATA = 0,
    PDU_ID_DIAGNOSTIC_STATUS,
    PDU_ID_ECU_STATUS,
    PDU_ID_FAULT_INJECTION_CMD,
    PDU_ID_OBD_REQUEST,
    PDU_ID_OBD_RESPONSE,
    PDU_ID_COUNT
} PduId_t;

typedef struct {
    PduId_t  pduId;
    uint32_t canId;
    uint8_t  dlc;
} CanIf_PduToCanIdMap_t;

extern const CanIf_PduToCanIdMap_t CanIf_PduMapTable[PDU_ID_COUNT];
```

**Hàm cần triển khai trong `CanIf.c`:**

| Hàm                        | Prototype                                                                           | Mô tả                                                                                                                            |
| --------------------------- | ----------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| CanIf_Init                  | `Std_ReturnType CanIf_Init(void);`                                                | Gọi Can_Init(), đăng ký Can_RegisterRxCallback(CanIf_RxCallback)                                                               |
| CanIf_Transmit              | `Std_ReturnType CanIf_Transmit(PduId_t pduId, const uint8_t *data, uint8_t len);` | Tra bảng CanIf_PduMapTable để lấy CAN ID, build CanFrame_t, gọi Can_Transmit()                                                |
| CanIf_RxCallback (internal) | `static void CanIf_RxCallback(const CanFrame_t *frame);`                          | Được Can module gọi khi có frame mới; tra ngược CAN ID → PduId; gọi`PduR_RxIndication(pduId, frame->data, frame->dlc)` |

### 3.3. File: `ServiceLayer/PduR/PduR.h` / `PduR.c`

| Hàm              | Prototype                                                                          | Mô tả                                                                             |
| ----------------- | ---------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| PduR_Transmit     | `Std_ReturnType PduR_Transmit(PduId_t pduId, const uint8_t *data, uint8_t len);` | Gọi thẳng`CanIf_Transmit()` (định tuyến tối giản vì chỉ có 1 kênh CAN) |
| PduR_RxIndication | `void PduR_RxIndication(PduId_t pduId, const uint8_t *data, uint8_t len);`       | Chuyển tiếp dữ liệu nhận được lên`Com_RxIndication()`                    |

---

## 4. `ECU_Firmware/ServiceLayer/` — Service Layer

### 4.1. File: `ServiceLayer/Com/Com.h` / `Com.c` + `Com_Cfg.h`

**Include:** `"Std_Types.h"`, `"PduR.h"`, `"SignalDefs.h"`, `"DtcList.h"`

**Struct dữ liệu signal nội bộ (Com Signal Buffer):**

```c
typedef struct {
    int16_t  coolantTempScaled;
    uint16_t oilPressureScaled;
    uint16_t batteryVoltageScaled;
    uint8_t  sensorFlags;
} Com_SensorSignals_t;

typedef struct {
    uint8_t confirmedCount;
    DtcId_t confirmedDtcs[3];
} Com_DiagnosticSignals_t;

typedef enum {
    ECU_STATE_INIT = 0,
    ECU_STATE_OK,
    ECU_STATE_DEGRADED,
    ECU_STATE_FAULT
} EcuState_t;

typedef struct {
    EcuState_t state;
    uint8_t    heartbeatCounter;
} Com_EcuStatusSignals_t;
```

**Hàm cần triển khai:**

| Hàm                                | Prototype                                                                    | Mô tả                                                                                                                                                                                                         |
| ----------------------------------- | ---------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Com_Init                            | `Std_ReturnType Com_Init(void);`                                           | Khởi tạo buffer signal nội bộ về giá trị mặc định                                                                                                                                                     |
| Com_WriteSensorSignals              | `void Com_WriteSensorSignals(const Com_SensorSignals_t *signals);`         | Ghi vào buffer, chờ MainFunction đóng gói và gửi theo chu kỳ                                                                                                                                            |
| Com_WriteDiagnosticSignals          | `void Com_WriteDiagnosticSignals(const Com_DiagnosticSignals_t *signals);` | Gọi khi DEM có cập nhật DTC, đánh dấu flag "cần gửi ngay"                                                                                                                                              |
| Com_WriteEcuStatusSignals           | `void Com_WriteEcuStatusSignals(const Com_EcuStatusSignals_t *signals);`   | Ghi trạng thái ECU vào buffer                                                                                                                                                                                |
| Com_MainFunction                    | `void Com_MainFunction(uint32_t currentTimeMs);`                           | Gọi định kỳ (ví dụ mỗi 10–50ms từ main loop); kiểm tra chu kỳ Tx của từng bản tin (100/500/1000ms) hoặc flag event-triggered, đóng gói payload theo`SignalDefs.h`, gọi `PduR_Transmit()` |
| Com_RxIndication                    | `void Com_RxIndication(PduId_t pduId, const uint8_t *data, uint8_t len);`  | Xử lý dữ liệu nhận (Fault Injection Command 0x400, OBD Request 0x500), giải mã và gọi callback tương ứng lên Application Layer qua RTE                                                             |
| Com_PackSensorData (internal)       | `static void Com_PackSensorData(uint8_t out[8]);`                          | Đóng gói`Com_SensorSignals_t` theo data layout 0x100                                                                                                                                                       |
| Com_PackDiagnosticStatus (internal) | `static void Com_PackDiagnosticStatus(uint8_t out[8]);`                    | Đóng gói theo data layout 0x200                                                                                                                                                                              |
| Com_PackEcuStatus (internal)        | `static void Com_PackEcuStatus(uint8_t out[4]);`                           | Đóng gói theo data layout 0x300                                                                                                                                                                              |

### 4.2. File: `ServiceLayer/Dem/Dem.h` / `Dem.c` + `Dem_Cfg.h`

**Include:** `"Std_Types.h"`, `"DtcList.h"`, `"NvM.h"`

**File `Dem_Cfg.h` — cấu hình EventId ↔ DTC, tham số debounce:**

```c
typedef enum {
    DEM_EVENT_COOLANT_OVERTEMP = 0,
    DEM_EVENT_OIL_PRESSURE_LOW,
    DEM_EVENT_BATTERY_UNDERVOLT,
    DEM_EVENT_COOLANT_INVALID,
    DEM_EVENT_SENSOR_TIMEOUT,
    DEM_EVENT_CAN_TIMEOUT,
    DEM_EVENT_COUNT
} DemEventId_t;

typedef enum {
    DEM_EVENT_STATUS_PASSED = 0,
    DEM_EVENT_STATUS_FAILED
} DemEventStatus_t;

#define DEM_DEBOUNCE_THRESHOLD   4   /* số chu kỳ liên tiếp để Confirm */

/* Bảng ánh xạ EventId -> DtcId (định nghĩa trong Dem_Cfg.c) */
extern const DtcId_t Dem_EventToDtcMap[DEM_EVENT_COUNT];
```

**Struct trạng thái DTC nội bộ:**

```c
typedef struct {
    uint8_t testFailed              : 1;
    uint8_t pending                 : 1;
    uint8_t confirmed               : 1;
    uint8_t testFailedSinceLastClear: 1;
    uint8_t reserved                : 4;
    uint8_t occurrenceCounter;
    uint8_t debounceCounter;   /* dùng nội bộ, không lưu NvM */
} DemDtcRecord_t;
```

**Hàm cần triển khai:**

| Hàm                          | Prototype                                                                             | Mô tả                                                                                                                                                                                                             |
| ----------------------------- | ------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Dem_Init                      | `Std_ReturnType Dem_Init(void);`                                                    | Khởi tạo bảng`DemDtcRecord_t dtcTable[DTC_ID_COUNT]`, gọi `NvM_ReadAll()` để khôi phục trạng thái sau reset                                                                                           |
| Dem_SetEventStatus            | `Std_ReturnType Dem_SetEventStatus(DemEventId_t eventId, DemEventStatus_t status);` | Được Diagnostic SWC gọi qua RTE mỗi chu kỳ; cập nhật debounceCounter tăng/giảm; nếu đạt`DEM_DEBOUNCE_THRESHOLD` → set confirmed=1, gọi `NvM_WriteBlock()`, gọi `Com_WriteDiagnosticSignals()` |
| Dem_GetDtcStatus              | `Std_ReturnType Dem_GetDtcStatus(DtcId_t dtcId, DemDtcRecord_t *record);`           | Trả về trạng thái hiện tại của 1 DTC (dùng cho OBD-II response)                                                                                                                                             |
| Dem_GetConfirmedDtcList       | `uint8_t Dem_GetConfirmedDtcList(DtcId_t outList[], uint8_t maxCount);`             | Trả về danh sách DTC đang Confirmed, trả về số lượng                                                                                                                                                       |
| Dem_ClearDtc                  | `Std_ReturnType Dem_ClearDtc(void);`                                                | Reset toàn bộ bảng DTC về mặc định, gọi`NvM_WriteBlock()` để xóa lưu trữ, thông báo COM cập nhật                                                                                                 |
| Dem_MainFunction (tùy chọn) | `void Dem_MainFunction(void);`                                                      | Nếu cần xử lý định kỳ riêng (ví dụ aging DTC) — có thể để trống trong scope rút gọn                                                                                                               |

### 4.3. File: `ServiceLayer/NvM/NvM.h` / `NvM.c` + `NvM_Cfg.h`

**Include:** `"Std_Types.h"`, `"Eeprom_I2C.h"`, `"DtcList.h"`

**File `NvM_Cfg.h`:**

```c
#define NVM_BLOCK_BASE_ADDR   0x0000u
#define NVM_BLOCK_SIZE        4u      /* byte/DTC: 1 status + 1 occurrence + 2 reserved */
#define NVM_MAGIC_NUMBER      0xA5u   /* đánh dấu block hợp lệ, tránh đọc rác lần đầu */
```

**Hàm cần triển khai:**

| Hàm           | Prototype                                                                       | Mô tả                                                                                                 |
| -------------- | ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| NvM_Init       | `Std_ReturnType NvM_Init(void);`                                              | Gọi`Eeprom_Init()`, kiểm tra Magic Number, nếu chưa có thì format block mặc định             |
| NvM_WriteBlock | `Std_ReturnType NvM_WriteBlock(DtcId_t dtcId, const DemDtcRecord_t *record);` | Tính địa chỉ =`NVM_BLOCK_BASE_ADDR + dtcId * NVM_BLOCK_SIZE`, ghi status/occurrence xuống EEPROM |
| NvM_ReadBlock  | `Std_ReturnType NvM_ReadBlock(DtcId_t dtcId, DemDtcRecord_t *record);`        | Đọc lại 1 block DTC                                                                                  |
| NvM_ReadAll    | `Std_ReturnType NvM_ReadAll(DemDtcRecord_t table[DTC_ID_COUNT]);`             | Đọc toàn bộ bảng DTC lúc khởi động (gọi trong`Dem_Init`)                                    |
| NvM_EraseAll   | `Std_ReturnType NvM_EraseAll(void);`                                          | Xóa toàn bộ block (dùng khi Clear DTC)                                                              |

---

## 5. `ECU_Firmware/Rte/` — Runtime Environment

### 5.1. File: `Rte/Rte.h` / `Rte.c` + `Rte_Cfg.h`

**Include:** tất cả header của Service Layer + Application Layer types (`"IoHwAb.h"`, `"Dem.h"`, `"Com.h"`)

**Nguyên tắc:** RTE không chứa logic nghiệp vụ, chỉ là các hàm "chuyển tiếp" đặt tên theo port Sender-Receiver (SR) / Client-Server (CS) để SWC không gọi thẳng Service Layer.

**Hàm cần triển khai (ví dụ các port cần thiết):**

| Hàm                                | Prototype                                                                                      | Mô tả                                                                                                                                     |
| ----------------------------------- | ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| Rte_Read_SensorData                 | `Std_ReturnType Rte_Read_SensorData(IoHwAb_SensorData_t *data);`                             | SWC gọi để đọc dữ liệu cảm biến mới nhất; bên trong gọi`IoHwAb_ReadAll()`                                                    |
| Rte_Write_SensorStatus              | `Std_ReturnType Rte_Write_SensorStatus(const SensorMonitoring_Status_t *status);`            | Sensor Monitoring SWC ghi kết quả giám sát; RTE lưu vào buffer nội bộ để Diagnostic SWC đọc                                     |
| Rte_Read_SensorStatus               | `Std_ReturnType Rte_Read_SensorStatus(SensorMonitoring_Status_t *status);`                   | Diagnostic SWC đọc lại kết quả giám sát                                                                                              |
| Rte_Call_Dem_SetEventStatus         | `Std_ReturnType Rte_Call_Dem_SetEventStatus(DemEventId_t eventId, DemEventStatus_t status);` | Diagnostic SWC gọi (Client-Server) tới`Dem_SetEventStatus()`                                                                            |
| Rte_Write_ComSensorSignals          | `void Rte_Write_ComSensorSignals(const Com_SensorSignals_t *signals);`                       | Sensor Monitoring SWC (hoặc RTE MainFunction) gọi để đẩy dữ liệu xuống`Com_WriteSensorSignals()`                                 |
| Rte_Call_FaultInjection_GetOverride | `bool Rte_Call_FaultInjection_GetOverride(DtcId_t relatedDtc, float *overrideValue);`        | Sensor Monitoring SWC hỏi xem có giá trị override đang active không (từ Fault Injection Handler)                                     |
| Rte_MainFunction                    | `void Rte_MainFunction(uint32_t currentTimeMs);`                                             | Gọi tuần tự các MainFunction cần thiết theo chu kỳ (SensorMonitoring, Diagnostic, Com, Dem) — đóng vai trò scheduler đơn giản |

---

## 6. `ECU_Firmware/Application/` — Application Layer

### 6.1. File: `Application/SensorMonitoring_SWC.h` / `.c`

**Include:** `"Rte.h"`, `"SignalDefs.h"`

**Struct/Enum trạng thái:**

```c
typedef enum {
    SENSOR_STATE_NORMAL = 0,
    SENSOR_STATE_OVER_RANGE,
    SENSOR_STATE_UNDER_RANGE,
    SENSOR_STATE_INVALID,
    SENSOR_STATE_TIMEOUT
} SensorState_t;

typedef struct {
    SensorState_t coolantState;
    SensorState_t oilState;
    SensorState_t batteryState;
    uint32_t      lastUpdateTimeMs;
} SensorMonitoring_Status_t;
```

**Hàm cần triển khai:**

| Hàm                                            | Prototype                                                                                                             | Mô tả                                                                                                                                                                                                                                    |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| SensorMonitoring_Init                           | `void SensorMonitoring_Init(void);`                                                                                 | Khởi tạo trạng thái mặc định NORMAL cho cả 3 cảm biến                                                                                                                                                                            |
| SensorMonitoring_MainFunction                   | `void SensorMonitoring_MainFunction(uint32_t currentTimeMs);`                                                       | Gọi định kỳ (≤100ms):`Rte_Read_SensorData()` → kiểm tra ngưỡng từng cảm biến (gọi 3 hàm Check bên dưới) → cập nhật `SensorMonitoring_Status_t` → `Rte_Write_SensorStatus()` → `Rte_Write_ComSensorSignals()` |
| SensorMonitoring_CheckCoolant (internal)        | `static SensorState_t SensorMonitoring_CheckCoolant(float tempC, SignalQuality_t quality);`                         | So sánh với`COOLANT_TEMP_MAX_C`, trả về state tương ứng                                                                                                                                                                           |
| SensorMonitoring_CheckOilPressure (internal)    | `static SensorState_t SensorMonitoring_CheckOilPressure(float pressureKpa, SignalQuality_t quality);`               | So sánh với`OIL_PRESSURE_MIN_KPA`                                                                                                                                                                                                      |
| SensorMonitoring_CheckBatteryVoltage (internal) | `static SensorState_t SensorMonitoring_CheckBatteryVoltage(float voltageV, SignalQuality_t quality);`               | So sánh với`BATTERY_VOLTAGE_MIN_V`                                                                                                                                                                                                     |
| SensorMonitoring_CheckTimeout (internal)        | `static bool SensorMonitoring_CheckTimeout(uint32_t lastUpdateTimeMs, uint32_t currentTimeMs, uint32_t timeoutMs);` | Kiểm tra timeout dựa trên timestamp cập nhật gần nhất                                                                                                                                                                               |

### 6.2. File: `Application/Diagnostic_SWC.h` / `.c`

**Include:** `"Rte.h"`, `"Dem_Cfg.h"`

**Hàm cần triển khai:**

| Hàm                                         | Prototype                                                                                   | Mô tả                                                                                                                                                                                                                                               |
| -------------------------------------------- | ------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Diagnostic_Init                              | `void Diagnostic_Init(void);`                                                             | Không cần state riêng, chỉ khởi tạo nếu cần                                                                                                                                                                                                   |
| Diagnostic_MainFunction                      | `void Diagnostic_MainFunction(uint32_t currentTimeMs);`                                   | Gọi định kỳ:`Rte_Read_SensorStatus()` → ánh xạ `SensorState_t` sang `DemEventId_t` tương ứng → gọi `Rte_Call_Dem_SetEventStatus()` cho từng event (Failed nếu state != NORMAL, Passed nếu NORMAL)                              |
| Diagnostic_MapSensorStateToEvents (internal) | `static void Diagnostic_MapSensorStateToEvents(const SensorMonitoring_Status_t *status);` | Logic ánh xạ chi tiết: OverRange Coolant → DEM_EVENT_COOLANT_OVERTEMP; UnderRange Oil → DEM_EVENT_OIL_PRESSURE_LOW; UnderRange Battery → DEM_EVENT_BATTERY_UNDERVOLT; Invalid → DEM_EVENT_COOLANT_INVALID; Timeout → DEM_EVENT_SENSOR_TIMEOUT |
| Diagnostic_CheckCanTimeout                   | `void Diagnostic_CheckCanTimeout(uint32_t currentTimeMs);`                                | Kiểm tra timeout truyền CAN (dùng ở phía GUI hoặc phía ECU nếu cần giám sát 2 chiều), gọi`Rte_Call_Dem_SetEventStatus(DEM_EVENT_CAN_TIMEOUT, ...)`                                                                                     |

### 6.3. File: `Application/FaultInjection_Handler.h` / `.c`

**Include:** `"Rte.h"`, `"DtcList.h"`

**Enum lệnh Fault Injection (khớp với CAN 0x400):**

```c
typedef enum {
    FI_ID_NONE = 0,
    FI_ID_COOLANT_OVERTEMP,
    FI_ID_OIL_PRESSURE_LOW,
    FI_ID_BATTERY_UNDERVOLT,
    FI_ID_SENSOR_INVALID,
    FI_ID_CAN_TIMEOUT
} FaultInjectionId_t;

typedef struct {
    bool    active;
    FaultInjectionId_t faultId;
    float   overrideValue;
} FaultInjection_State_t;
```

**Hàm cần triển khai:**

| Hàm                         | Prototype                                                                                   | Mô tả                                                                                                                       |
| ---------------------------- | ------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| FaultInjection_Init          | `void FaultInjection_Init(void);`                                                         | Reset trạng thái active = false                                                                                             |
| FaultInjection_HandleCommand | `void FaultInjection_HandleCommand(uint8_t faultId, uint8_t action);`                     | Được`Com_RxIndication` gọi (qua RTE) khi nhận bản tin 0x400; set/clear trạng thái override tương ứng             |
| FaultInjection_GetOverride   | `bool FaultInjection_GetOverride(FaultInjectionId_t relatedFault, float *overrideValue);` | Được Sensor Monitoring SWC gọi qua`Rte_Call_FaultInjection_GetOverride()` để biết có cần ép giá trị giả không |
| FaultInjection_ClearAll      | `void FaultInjection_ClearAll(void);`                                                     | Xóa toàn bộ override đang active (faultId = 0, action = 0)                                                                |

### 6.4. File: `main.c` (ECU)

**Include:** tất cả header ở trên

**Hàm cần triển khai:**

| Hàm | Prototype           | Mô tả                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| ---- | ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| main | `int main(void);` | Gọi`HAL_Init()`, cấu hình clock, gọi `Adc_Init()`, `Can_Init()`, `Eeprom_Init()`, `IoHwAb_Init()`, `CanIf_Init()`, `PduR` (không cần init riêng), `Com_Init()`, `NvM_Init()`, `Dem_Init()`, `SensorMonitoring_Init()`, `Diagnostic_Init()`, `FaultInjection_Init()`, `Rte` (không cần init riêng nếu không giữ state); sau đó vào vòng lặp `while(1)` gọi `Rte_MainFunction(HAL_GetTick())` |

---

## 7. `GUI_Firmware/` — Firmware GUI (Thành viên 2)

### 7.1. File: `Display/Ssd1306_Driver.h` / `.c`

**Include:** `stm32f1xx_hal_i2c.h`

| Hàm             | Prototype                                                          | Mô tả                                                                                                |
| ---------------- | ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| Ssd1306_Init     | `Std_ReturnType Ssd1306_Init(void);`                             | Khởi tạo I2C + gửi lệnh init chuẩn cho SSD1306                                                    |
| Ssd1306_Clear    | `void Ssd1306_Clear(void);`                                      | Xóa buffer màn hình                                                                                 |
| Ssd1306_DrawText | `void Ssd1306_DrawText(uint8_t x, uint8_t y, const char *text);` | Vẽ chuỗi ký tự tại tọa độ (có thể dùng thư viện có sẵn u8g2/SSD1306 thay vì tự viết) |
| Ssd1306_Update   | `void Ssd1306_Update(void);`                                     | Đẩy buffer ra màn hình vật lý (I2C transfer)                                                     |

### 7.2. File: `Display/Screen_SensorDashboard.h` / `.c` (và tương tự cho 4 screen khác)

| Hàm                             | Prototype                                                                 | Mô tả                                        |
| -------------------------------- | ------------------------------------------------------------------------- | ---------------------------------------------- |
| Screen_SensorDashboard_Render    | `void Screen_SensorDashboard_Render(const Com_SensorSignals_t *data);`  | Vẽ giá trị 3 cảm biến lên OLED           |
| Screen_EcuStatus_Render          | `void Screen_EcuStatus_Render(const Com_EcuStatusSignals_t *status);`   | Vẽ trạng thái ECU + heartbeat               |
| Screen_DtcDisplay_Render         | `void Screen_DtcDisplay_Render(const DtcId_t *dtcList, uint8_t count);` | Vẽ danh sách DTC (tra tên từ`Dtc_Table`) |
| Screen_CanStatus_Render          | `void Screen_CanStatus_Render(bool connected, uint32_t rxCount);`       | Vẽ trạng thái kết nối CAN                 |
| Screen_FaultInjectionMenu_Render | `void Screen_FaultInjectionMenu_Render(uint8_t selectedIndex);`         | Vẽ menu chọn lỗi để inject                |

### 7.3. File: `Input/Button_Driver.h` / `.c`

```c
typedef enum {
    BUTTON_NONE = 0,
    BUTTON_UP,
    BUTTON_DOWN,
    BUTTON_SELECT
} ButtonEvent_t;
```

| Hàm        | Prototype                            | Mô tả                                                                                                              |
| ----------- | ------------------------------------ | -------------------------------------------------------------------------------------------------------------------- |
| Button_Init | `void Button_Init(void);`          | Cấu hình GPIO input (pull-up), có thể dùng EXTI hoặc polling                                                   |
| Button_Poll | `ButtonEvent_t Button_Poll(void);` | Đọc trạng thái nút, có debounce phần mềm (ví dụ kiểm tra ổn định trong 20ms), trả về sự kiện nhấn |

### 7.4. File: `Input/MenuStateMachine.h` / `.c`

```c
typedef enum {
    MENU_SCREEN_SENSOR_DASHBOARD = 0,
    MENU_SCREEN_ECU_STATUS,
    MENU_SCREEN_DTC_DISPLAY,
    MENU_SCREEN_CAN_STATUS,
    MENU_SCREEN_FAULT_INJECTION,
    MENU_SCREEN_COUNT
} MenuScreen_t;

typedef struct {
    MenuScreen_t currentScreen;
    uint8_t      faultInjectionSelectedIndex;
} MenuState_t;
```

| Hàm                          | Prototype                                                                        | Mô tả                                                                                                                                                                 |
| ----------------------------- | -------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| MenuStateMachine_Init         | `void MenuStateMachine_Init(MenuState_t *state);`                              | Khởi tạo màn hình mặc định = MENU_SCREEN_SENSOR_DASHBOARD                                                                                                        |
| MenuStateMachine_HandleButton | `void MenuStateMachine_HandleButton(MenuState_t *state, ButtonEvent_t event);` | UP/DOWN chuyển giữa các`MenuScreen_t`; nếu đang ở FAULT_INJECTION thì UP/DOWN di chuyển selectedIndex, SELECT thì gọi `CanIf_Gui_SendFaultInjectionCmd()` |

### 7.5. File: `Can/Can_Gui.h`/`.c`, `Can/CanIf_Gui.h`/`.c`

(Tái sử dụng thiết kế `Can.h`/`CanIf.h` từ ECU_Firmware nếu cùng họ MCU — chỉ khác các hàm xử lý Rx tương ứng phía GUI)

| Hàm                            | Prototype                                                                            | Mô tả                                                                                                       |
| ------------------------------- | ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------- |
| CanIf_Gui_Init                  | `Std_ReturnType CanIf_Gui_Init(void);`                                             | Giống CanIf_Init phía ECU nhưng đăng ký callback xử lý Rx cho 0x100/0x200/0x300                       |
| CanIf_Gui_RxCallback (internal) | `static void CanIf_Gui_RxCallback(const CanFrame_t *frame);`                       | Giải mã theo CAN ID nhận được, cập nhật buffer dữ liệu global để`main.c` render lên màn hình |
| CanIf_Gui_SendFaultInjectionCmd | `Std_ReturnType CanIf_Gui_SendFaultInjectionCmd(uint8_t faultId, uint8_t action);` | Đóng gói và gửi bản tin 0x400                                                                           |
| CanIf_Gui_CheckTimeout          | `bool CanIf_Gui_CheckTimeout(uint32_t currentTimeMs);`                             | Kiểm tra nếu không nhận được bản tin 0x100 trong >200ms (2x chu kỳ) → coi là CAN timeout           |

### 7.6. File: `main.c` (GUI)

| Hàm | Prototype           | Mô tả                                                                                                                                                                                                                                                                         |
| ---- | ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| main | `int main(void);` | Gọi`HAL_Init()`, `Ssd1306_Init()`, `Button_Init()`, `CanIf_Gui_Init()`, `MenuStateMachine_Init()`; vòng lặp `while(1)`: `Button_Poll()` → `MenuStateMachine_HandleButton()` → gọi hàm Render tương ứng với `currentScreen` → `Ssd1306_Update()` |

---

## 8. `OBD_Reader_Tool/` — Công cụ đọc OBD-II (Python, nếu dùng PC + USB-CAN)

### 8.1. File: `obd_reader.py`

**Import:** `can` (thư viện python-can), `time`

**Hàm cần triển khai:**

| Hàm                   | Prototype (Python)                                                            | Mô tả                                                                   |
| ---------------------- | ----------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| init_can_bus           | `def init_can_bus(channel: str, bitrate: int = 500000) -> can.Bus`          | Khởi tạo kết nối tới USB-CAN adapter                                 |
| send_read_dtc_request  | `def send_read_dtc_request(bus: can.Bus) -> None`                           | Gửi frame CAN ID 0x500, mode=0x03                                        |
| send_clear_dtc_request | `def send_clear_dtc_request(bus: can.Bus) -> None`                          | Gửi frame CAN ID 0x500, mode=0x04                                        |
| receive_dtc_response   | `def receive_dtc_response(bus: can.Bus, timeout: float = 1.0) -> list[int]` | Chờ nhận frame CAN ID 0x501, giải mã danh sách mã DTC               |
| decode_dtc_response    | `def decode_dtc_response(payload: bytes) -> list[int]`                      | Tách các mã DTC 2-byte từ payload theo data layout mục 12.3          |
| main                   | `def main() -> None`                                                        | CLI đơn giản: menu chọn Read DTC / Clear DTC, in kết quả ra console |

---

## 9. Bảng tổng hợp State Machine cần triển khai

| State Machine         | File chứa                   | Các trạng thái                                                                                                                                                             |
| --------------------- | ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Sensor State          | `SensorMonitoring_SWC.h`   | NORMAL → OVER_RANGE / UNDER_RANGE / INVALID / TIMEOUT                                                                                                                        |
| DTC Status (per DTC)  | `Dem.h`                    | (chưa phát hiện) → Pending (debounce đang đếm) → Confirmed (đã lưu NvM) → Passed (debounce ngược về 0, TestFailed=0 nhưng TestFailedSinceLastClear vẫn giữ) |
| ECU State             | `Com.h` (`EcuState_t`)   | INIT → OK → DEGRADED (có DTC Pending) → FAULT (có DTC Confirmed)                                                                                                         |
| GUI Menu State        | `MenuStateMachine.h`       | SENSOR_DASHBOARD ↔ ECU_STATUS ↔ DTC_DISPLAY ↔ CAN_STATUS ↔ FAULT_INJECTION (chuyển bằng UP/DOWN)                                                                        |
| Fault Injection State | `FaultInjection_Handler.h` | INACTIVE → ACTIVE (override giá trị cảm biến) → INACTIVE (khi Clear)                                                                                                    |

---

## 10. Checklist triển khai theo thành viên (tham chiếu nhanh)

### Thành viên 1 — cần code các file sau

- `Mcal/Adc/*`, `Mcal/Dio/*` (nếu dùng), `Mcal/Eeprom/*`
- `EcuAbstraction/IoHwAb/*`
- `ServiceLayer/Dem/*`, `ServiceLayer/NvM/*`
- `Application/SensorMonitoring_SWC.*`, `Application/Diagnostic_SWC.*`
- `Rte/*` (phối hợp cùng Thành viên 2 vì RTE kết nối cả 2 phía Application/Com)

### Thành viên 2 — cần code các file sau

- `Mcal/Can/*`
- `EcuAbstraction/CanIf/*`, `ServiceLayer/PduR/*`, `ServiceLayer/Com/*`
- Toàn bộ `GUI_Firmware/*`
- `OBD_Reader_Tool/obd_reader.py`
- `Application/FaultInjection_Handler.*` (vì liên quan trực tiếp lệnh CAN 0x400 từ GUI)

### Chung cả 2 người

- `Common/CanIds.h`, `Common/DtcList.h/.c`, `Common/SignalDefs.h`, `Common/Std_Types.h` — **PHẢI thống nhất và khóa (freeze) nội dung các file này trước Tuần 3**, vì mọi module khác đều phụ thuộc vào đây.
- `main.c` của mỗi node (mỗi người tự viết cho node mình phụ trách nhưng theo đúng thứ tự khởi tạo đã liệt kê ở mục 6.4/7.6).

---

*File này là đặc tả mức thiết kế chi tiết (low-level design), dùng trực tiếp làm cơ sở để code. Tên hàm/struct có thể điều chỉnh nhỏ khi triển khai thực tế (ví dụ theo IDE auto-complete hoặc thư viện HAL cụ thể của MCU), nhưng cần giữ đúng vai trò và luồng gọi giữa các lớp như đã mô tả để đảm bảo nhất quán với kiến trúc AUTOSAR đã chốt trong đề cương.*
