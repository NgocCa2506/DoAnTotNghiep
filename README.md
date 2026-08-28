# ĐỀ CƯƠNG ĐỒ ÁN TỐT NGHIỆP

## Thiết kế và triển khai hệ thống giám sát cảm biến trên ô tô theo tiêu chuẩn AUTOSAR

**Nhóm thực hiện:** 2 sinh viên
**Thời gian thực hiện:** 12 tuần (~3 tháng)

---

## 1. Tên đề tài

**Thiết kế và triển khai hệ thống giám sát cảm biến trên ô tô theo tiêu chuẩn AUTOSAR (Prototype)**

Tên tiếng Anh gợi ý: *AUTOSAR-based Automotive Sensor Monitoring and Diagnostic System (Prototype)*

---

## 2. Tổng quan

Đồ án xây dựng một **ECU prototype** dựa trên vi điều khiển (MCU), có nhiệm vụ đọc dữ liệu từ các cảm biến ô tô mô phỏng, giám sát và phát hiện bất thường, quản lý sự kiện chẩn đoán (Diagnostic Event) và mã lỗi (DTC), lưu trạng thái lỗi vào bộ nhớ không bay hơi (NVM), và truyền dữ liệu/trạng thái ra ngoài qua giao thức **CAN**. Một thiết bị GUI (màn hình OLED I2C, điều khiển bằng nút bấm) đóng vai trò màn hình giám sát, nhận dữ liệu từ ECU qua CAN để hiển thị cảm biến, trạng thái ECU và lỗi. Ngoài ra, hệ thống hỗ trợ xuất chẩn đoán theo chuẩn **OBD-II** ở mức đơn giản và một bộ đọc OBD-II tối giản để đọc lại các mã lỗi này.

Kiến trúc phần mềm trên MCU được tổ chức **theo tư duy AUTOSAR Classic** (không triển khai toàn bộ chuẩn AUTOSAR đầy đủ), gồm các lớp: Application, RTE, Service Layer (DEM, NvM, COM, PduR), ECU Abstraction Layer (CanIf, IoHwAb), và MCAL (CAN Driver, ADC, DIO). Cách tiếp cận này giúp sinh viên hiểu và thể hiện được tư duy phân lớp, tách biệt phần cứng và phần mềm ứng dụng theo đúng triết lý AUTOSAR, trong khi vẫn giữ khối lượng công việc khả thi trong 3 tháng với 2 người.

Đồ án tập trung vào **luồng lõi**:

```
Sensor → ECU (MCAL/IoHwAb/RTE/SWC) → Fault Diagnosis → DEM/DTC → CAN → GUI
                                                              ↘ OBD-II export → OBD-II Reader
```

---

## 3. Mục tiêu

1. Đọc dữ liệu cảm biến (ADC/DIO) trên MCU.
2. Giám sát giá trị cảm biến (ngưỡng, dải hợp lệ, timeout).
3. Phát hiện lỗi/bất thường (over-range, under-range, invalid signal, sensor timeout, CAN timeout).
4. Tạo Diagnostic Event khi phát hiện bất thường.
5. Quản lý DTC (status byte, occurrence, healing/aging đơn giản).
6. Lưu trạng thái lỗi (DTC) vào NVM khi phù hợp.
7. Gửi dữ liệu cảm biến qua CAN theo chu kỳ.
8. Gửi trạng thái lỗi/DTC qua CAN khi có thay đổi.
9. Nhận yêu cầu điều khiển/fault injection từ GUI qua CAN.
10. GUI hiển thị dữ liệu cảm biến theo thời gian thực.
11. GUI hiển thị trạng thái ECU (OK/Degraded/Fault).
12. GUI hiển thị danh sách DTC/lỗi hiện hành.
13. Có cơ chế Fault Injection để tạo tình huống lỗi phục vụ demo.
14. Kiểm thử được toàn bộ luồng end-to-end: Sensor → ECU → Diagnosis → DTC → CAN → GUI.
15. Xuất được chẩn đoán lỗi theo khuôn mẫu đơn giản hóa của OBD-II và đọc lại được bằng bộ đọc riêng.

---

## 4. Phạm vi

### 4.1. Trong phạm vi (Core Scope)

- 1 MCU đóng vai trò ECU prototype, đọc trực tiếp cảm biến qua ADC/DIO.
- Kiến trúc phần mềm phân lớp theo tư duy AUTOSAR Classic (rút gọn, không dùng công cụ cấu hình AUTOSAR thương mại như Vector DaVinci).
- Giám sát 3–4 cảm biến, phát hiện lỗi theo ngưỡng/dải/timeout.
- DEM quản lý một số lượng DTC vừa đủ (~5–6 mã).
- Lưu DTC vào NVM (mô phỏng bằng EEPROM/Flash nội bộ MCU hoặc file mô phỏng, tùy nền tảng).
- Giao tiếp CAN giữa MCU và PC/thiết bị GUI (dùng CAN transceiver + module CAN-USB hoặc 2 MCU có CAN).
- GUI trên màn hình OLED I2C + nút bấm điều hướng, chỉ hiển thị dữ liệu ECU gửi về.
- Fault Injection đơn giản (qua nút bấm/lệnh từ GUI hoặc lệnh CAN).
- Xuất bản tin chẩn đoán dạng đơn giản hóa theo tinh thần OBD-II (không phải UDS/ISO 14229 đầy đủ) và một bộ đọc OBD-II tối giản (có thể là chương trình trên PC hoặc MCU thứ hai).
- Test plan và kịch bản demo end-to-end.

### 4.2. Ngoài phạm vi bắt buộc (không tự ý mở rộng thành yêu cầu chính)

Các mục sau **không** được coi là chức năng bắt buộc của core project. Nếu đề xuất, chỉ được xếp vào mục 22 – *Optional/Future Work*, kèm giải thích lý do không cần thiết:

- Diagnostic Tester chuyên dụng độc lập.
- Máy chẩn đoán ô tô chuyên dụng (scan tool thương mại).
- UDS đầy đủ (ISO 14229) với đầy đủ dịch vụ (Session Control, Security Access, Routine Control...).
- Gateway ECU (định tuyến giữa nhiều mạng CAN/LIN/Ethernet).
- Bootloader và ECU reprogramming (flashing qua CAN).
- Triển khai toàn bộ chuẩn AUTOSAR Classic (Basic Software đầy đủ, RTE Generator tự động, cấu hình ARXML chuẩn).
- Bất kỳ chức năng nào không phục vụ trực tiếp mục tiêu 1–15 ở mục 3.

**Lưu ý quan trọng:** GUI trong đồ án **không mặc định được gọi là Diagnostic Tester**. GUI trước hết chỉ đóng vai trò là **màn hình giám sát và hiển thị** (monitoring display), nhận dữ liệu thụ động từ ECU qua CAN. Việc GUI gửi lệnh Fault Injection là một chức năng phụ trợ phục vụ demo, không biến GUI thành một Diagnostic Tester theo đúng nghĩa UDS.

---

## 5. Kiến trúc tổng thể

```
┌─────────────────────────────┐         CAN Bus         ┌─────────────────────────────┐
│         ECU (MCU)           │◄────────────────────────►│      GUI Device / PC        │
│                              │                          │                              │
│  Sensors → ADC/DIO           │                          │  OLED I2C Display            │
│  → Sensor Monitoring SWC     │                          │  Nút bấm điều hướng          │
│  → Diagnostic SWC            │                          │  Nhận CAN → hiển thị:        │
│  → DEM → DTC                 │                          │   - Sensor Dashboard         │
│  → NvM (lưu DTC)              │                          │   - ECU Status               │
│  → COM/PduR/CanIf/CAN Driver │                          │   - DTC List                 │
│                              │                          │   - CAN Status                │
└──────────────┬───────────────┘                          └─────────────────────────────┘
               │
               │ OBD-II style export (qua CAN, request/response đơn giản)
               ▼
      ┌─────────────────────┐
      │   OBD-II Reader       │
      │ (PC tool / MCU phụ)   │
      │ Đọc DTC theo yêu cầu   │
      └─────────────────────┘
```

**Luồng dữ liệu chính:**

```
Sensor (Coolant/Oil/Battery/...) 
   → ADC/DIO (MCAL) 
   → IoHwAb 
   → Sensor Monitoring SWC (qua RTE) 
   → Diagnostic SWC 
   → DEM 
   → DTC (+ NvM lưu trạng thái) 
   → COM → PduR → CanIf → CAN Driver 
   → CAN Bus 
   → GUI (hiển thị) / OBD-II Reader (đọc DTC theo yêu cầu)
```

---

## 6. Kiến trúc AUTOSAR (rút gọn, phù hợp đồ án sinh viên)

Đồ án áp dụng **tư duy phân lớp của AUTOSAR Classic Platform**, không dùng công cụ cấu hình ARXML thương mại, mà tự hiện thực các lớp bằng C, mô phỏng đúng vai trò và giao diện (interface) giữa các lớp.

| Lớp | Vai trò trong đồ án | Mức triển khai |
|---|---|---|
| **Application Layer** | Chứa các SWC nghiệp vụ: Sensor Monitoring SWC, Diagnostic SWC | Triển khai thật, là trọng tâm đồ án |
| **RTE (Runtime Environment)** | Lớp trung gian, chuẩn hóa giao tiếp giữa SWC và Service Layer bằng các hàm kiểu Sender-Receiver/Client-Server rút gọn | Triển khai thật nhưng viết tay (hand-written RTE), không dùng RTE Generator |
| **Service Layer** | DEM, NvM, COM, PduR | Triển khai thật ở mức rút gọn (simplified) |
| **ECU Abstraction Layer** | IoHwAb, CanIf | Triển khai thật ở mức rút gọn |
| **MCAL** | CAN Driver, ADC Driver, DIO Driver | Dùng driver có sẵn của hãng MCU (HAL/SDK), được "bọc" lại theo giao diện kiểu MCAL |

Việc này giúp thể hiện đúng **nguyên lý phân tách phần cứng – phần mềm ứng dụng** của AUTOSAR mà không cần bộ công cụ AUTOSAR thương mại, vốn không khả thi trong 3 tháng với 2 sinh viên.

---

## 7. Mô tả từng module

Với mỗi module: chức năng, input/output, quan hệ gọi, luồng dữ liệu, mức triển khai (thật/mock), mức ưu tiên.

### 7.1. Sensor Monitoring SWC (Application Layer)

| Mục | Nội dung |
|---|---|
| Chức năng | Nhận giá trị cảm biến thô đã quy đổi từ IoHwAb (qua RTE), kiểm tra ngưỡng/dải hợp lệ/timeout, xác định trạng thái cảm biến (Normal/Over/Under/Invalid/Timeout) |
| Input | Giá trị cảm biến (qua RTE_Read từ IoHwAb) |
| Output | Trạng thái giám sát cảm biến (gửi sang Diagnostic SWC qua RTE), giá trị cảm biến để gửi CAN (qua COM) |
| Gọi / được gọi | Được lập lịch định kỳ (task scheduler); gọi RTE để đọc IoHwAb; gọi RTE để gửi kết quả cho Diagnostic SWC và COM |
| Luồng dữ liệu | ADC/DIO → IoHwAb → RTE → Sensor Monitoring SWC → RTE → Diagnostic SWC / COM |
| Mock hay thật | Triển khai thật (đây là module lõi) |
| Ưu tiên | **Cao** |

### 7.2. Diagnostic SWC (Application Layer)

| Mục | Nội dung |
|---|---|
| Chức năng | Nhận trạng thái lỗi từ Sensor Monitoring SWC (và từ module giám sát CAN timeout), quyết định tạo Diagnostic Event, gọi DEM để báo cáo lỗi (Fail/Pass) |
| Input | Trạng thái cảm biến (qua RTE); trạng thái CAN timeout |
| Output | Lệnh gọi DEM_SetEventStatus(EventId, Status) |
| Gọi / được gọi | Được gọi bởi RTE sau khi Sensor Monitoring SWC cập nhật; gọi RTE → DEM |
| Luồng dữ liệu | Sensor Monitoring SWC → RTE → Diagnostic SWC → RTE → DEM |
| Mock hay thật | Triển khai thật |
| Ưu tiên | **Cao** |

### 7.3. RTE (Runtime Environment)

| Mục | Nội dung |
|---|---|
| Chức năng | Cung cấp các hàm giao tiếp chuẩn hóa (Rte_Read/Rte_Write/Rte_Call) giữa SWC và Service Layer, che giấu chi tiết triển khai bên dưới |
| Input/Output | Là lớp trung gian, không tự sinh dữ liệu, chỉ chuyển tiếp |
| Gọi / được gọi | Được gọi bởi tất cả SWC; gọi xuống DEM, NvM, COM, IoHwAb |
| Luồng dữ liệu | Trung chuyển hai chiều giữa Application Layer và Service/ECU Abstraction Layer |
| Mock hay thật | Triển khai thật nhưng viết tay đơn giản (không dùng RTE Generator) |
| Ưu tiên | **Trung bình–Cao** (cần có để giữ đúng tư duy AUTOSAR, nhưng có thể đơn giản hóa tối đa) |

### 7.4. DEM (Diagnostic Event Manager)

| Mục | Nội dung |
|---|---|
| Chức năng | Quản lý các Diagnostic Event, ánh xạ Event → DTC, quản lý status byte của DTC (TestFailed, Confirmed, Pending...), gọi NvM để lưu khi cần |
| Input | DEM_SetEventStatus(EventId, Status) từ Diagnostic SWC |
| Output | Cập nhật bảng DTC status; gửi thông báo cho COM để phát CAN; gọi NvM_WriteBlock khi DTC được confirm |
| Gọi / được gọi | Được gọi bởi RTE (từ Diagnostic SWC); gọi NvM và thông báo cho COM (qua RTE) |
| Luồng dữ liệu | Diagnostic SWC → RTE → DEM → (NvM, COM) |
| Mock hay thật | Triển khai thật ở mức rút gọn (không cần đầy đủ debounce counter phức tạp như AUTOSAR chuẩn, có thể dùng bộ đếm đơn giản) |
| Ưu tiên | **Cao** |

### 7.5. NvM (NVRAM Manager)

| Mục | Nội dung |
|---|---|
| Chức năng | Lưu/đọc dữ liệu DTC (status, số lần xảy ra) vào bộ nhớ không bay hơi |
| Input | Yêu cầu ghi/đọc từ DEM |
| Output | Dữ liệu DTC được lưu bền vững, khôi phục sau reset |
| Gọi / được gọi | Được gọi bởi DEM; gọi driver EEPROM/Flash nội bộ MCU |
| Luồng dữ liệu | DEM → NvM → EEPROM/Flash (MCAL) |
| Mock hay thật | Triển khai thật ở mức đơn giản (dùng EEPROM mô phỏng bằng Flash sector hoặc EEPROM nội bộ MCU nếu có) |
| Ưu tiên | **Trung bình** |

### 7.6. COM (Communication Manager)

| Mục | Nội dung |
|---|---|
| Chức năng | Đóng gói dữ liệu (sensor value, DTC status, ECU status) thành các tín hiệu (signal) và ghép vào PDU theo chu kỳ hoặc theo sự kiện |
| Input | Giá trị cảm biến, trạng thái DTC, trạng thái ECU (qua RTE) |
| Output | PDU gửi xuống PduR; dữ liệu nhận từ PduR được giải mã thành signal cho RTE |
| Gọi / được gọi | Được gọi bởi RTE (Tx path); gọi PduR (Tx); được PduR gọi callback khi nhận (Rx path) |
| Luồng dữ liệu | RTE → COM → PduR (Tx); PduR → COM → RTE (Rx) |
| Mock hay thật | Triển khai thật ở mức rút gọn (mapping signal cố định, không cần cấu hình động) |
| Ưu tiên | **Cao** |

### 7.7. PduR (PDU Router)

| Mục | Nội dung |
|---|---|
| Chức năng | Định tuyến PDU giữa COM và CanIf (trong đồ án chỉ có 1 tuyến CAN nên định tuyến đơn giản, mang tính minh họa kiến trúc) |
| Input/Output | PDU từ COM → CanIf; PDU từ CanIf → COM |
| Gọi / được gọi | Gọi bởi COM và CanIf hai chiều |
| Luồng dữ liệu | COM ↔ PduR ↔ CanIf |
| Mock hay thật | Có thể triển khai tối giản (gần như pass-through) vì chỉ có 1 kênh CAN |
| Ưu tiên | **Thấp–Trung bình** (giữ để đúng kiến trúc, nhưng không cần định tuyến phức tạp) |

### 7.8. CanIf (CAN Interface)

| Mục | Nội dung |
|---|---|
| Chức năng | Trừu tượng hóa CAN Driver, ánh xạ CAN ID ↔ PDU ID, cung cấp API chuẩn hóa cho lớp trên |
| Input/Output | PDU từ PduR → CAN frame (Tx); CAN frame → PDU cho PduR (Rx) |
| Gọi / được gọi | Gọi bởi PduR; gọi CAN Driver |
| Luồng dữ liệu | PduR ↔ CanIf ↔ CAN Driver |
| Mock hay thật | Triển khai thật ở mức rút gọn |
| Ưu tiên | **Trung bình** |

### 7.9. CAN Driver (MCAL)

| Mục | Nội dung |
|---|---|
| Chức năng | Điều khiển phần cứng CAN controller: cấu hình, gửi/nhận frame, ngắt |
| Input/Output | CAN frame vật lý trên bus |
| Gọi / được gọi | Gọi bởi CanIf; tương tác trực tiếp với phần cứng CAN peripheral |
| Luồng dữ liệu | CanIf ↔ CAN Driver ↔ CAN Controller (HW) |
| Mock hay thật | Dùng driver/SDK có sẵn của hãng MCU, bọc lại theo API MCAL | 
| Ưu tiên | **Cao** (bắt buộc để có CAN thật) |

### 7.10. IoHwAb (I/O Hardware Abstraction)

| Mục | Nội dung |
|---|---|
| Chức năng | Trừu tượng hóa việc đọc cảm biến (ADC/DIO), quy đổi giá trị thô sang đơn vị vật lý (°C, V, %) |
| Input | Giá trị thô từ ADC/DIO Driver |
| Output | Giá trị cảm biến đã quy đổi, cung cấp cho RTE |
| Gọi / được gọi | Gọi bởi RTE (qua Sensor Monitoring SWC); gọi ADC/DIO Driver |
| Luồng dữ liệu | ADC/DIO → IoHwAb → RTE → Sensor Monitoring SWC |
| Mock hay thật | Triển khai thật (đơn giản, hàm quy đổi tuyến tính hoặc bảng tra) |
| Ưu tiên | **Cao** |

### 7.11. ADC Driver (MCAL)

| Mục | Nội dung |
|---|---|
| Chức năng | Đọc điện áp analog từ cảm biến (Coolant Temp qua NTC, Oil Pressure qua cảm biến áp suất analog, Battery Voltage qua chia áp) |
| Input/Output | Điện áp analog → giá trị số (raw ADC value) |
| Gọi / được gọi | Gọi bởi IoHwAb | 
| Mock hay thật | Dùng SDK/HAL có sẵn của MCU |
| Ưu tiên | **Cao** |

### 7.12. DIO Driver (MCAL)

| Mục | Nội dung |
|---|---|
| Chức năng | Đọc tín hiệu số (ví dụ Door Status: open/closed) |
| Input/Output | Mức điện áp digital → giá trị 0/1 |
| Gọi / được gọi | Gọi bởi IoHwAb |
| Mock hay thật | Dùng SDK/HAL có sẵn của MCU |
| Ưu tiên | **Trung bình** (nếu chọn cảm biến Door Status; có thể bỏ nếu không dùng) |

---

## 8. Thiết kế cảm biến

### 8.1. Số lượng đề xuất: **3 cảm biến analog + 1 màn hình OLED GUI**

| Cảm biến | Loại tín hiệu | Lý do chọn |
|---|---|---|
| **Coolant Temperature** | Analog (mô phỏng bằng NTC thermistor hoặc biến trở + ADC) | Dễ mô phỏng bằng biến trở, có ngưỡng lỗi rõ ràng (over-temperature), là kịch bản lỗi kinh điển trong ô tô |
| **Oil Pressure** | Analog (mô phỏng bằng biến trở hoặc cảm biến áp suất giá rẻ) | Dễ mô phỏng, minh họa lỗi under-range (áp suất thấp) |
| **Battery Voltage** | Analog (đọc trực tiếp qua mạch chia áp từ nguồn cấp board) | Không cần cảm biến rời, chỉ cần mạch chia áp đơn giản, minh họa lỗi under-voltage |
| **OLED I2C Display** | I2C (không phải cảm biến, nhưng là thiết bị hiển thị GUI bắt buộc) | Cần thiết cho toàn bộ chức năng GUI |

**Lý do không chọn Fuel Level và Door Status:** Với 2 người trong 3 tháng, việc thêm cảm biến không làm tăng độ khó kỹ thuật cốt lõi (đều là ADC/DIO) nhưng làm tăng thời gian tích hợp, dây nối, hiệu chỉnh ngưỡng và mở rộng số lượng DTC/CAN signal cần quản lý — không mang lại giá trị học thuật tương xứng với chi phí thời gian. 3 cảm biến analog là đủ để minh họa đầy đủ các loại lỗi (over-range, under-range, invalid, timeout) mà không làm phình to hệ thống.

Nếu còn dư thời gian ở Phase 9, có thể bổ sung Door Status (DIO) như một cảm biến phụ — xem mục 22.

---

## 9. Thiết kế Fault Diagnosis

### 9.1. Luồng phát hiện lỗi

```
Sensor Value (raw)
   ↓  ADC/DIO → IoHwAb (quy đổi đơn vị)
Physical Value
   ↓  Sensor Monitoring SWC
Threshold / Range / Timeout Check
   ↓
Sensor Status (Normal / OverRange / UnderRange / Invalid / Timeout)
   ↓  Diagnostic SWC
Diagnostic Event (Failed/Passed)
   ↓  DEM_SetEventStatus()
DEM (cập nhật status byte, debounce counter đơn giản)
   ↓  Nếu đủ điều kiện xác nhận (confirmed)
DTC (mã lỗi cụ thể) → NvM lưu → COM/PduR/CanIf/CAN Driver phát lên CAN
```

### 9.2. Các loại lỗi được phát hiện

| Loại lỗi | Điều kiện | Áp dụng cho |
|---|---|---|
| Over-range | Giá trị > ngưỡng trên | Coolant Temp |
| Under-range | Giá trị < ngưỡng dưới | Oil Pressure, Battery Voltage |
| Invalid signal | Giá trị ADC ngoài dải vật lý hợp lệ (ví dụ đứt/chập cảm biến: 0V hoặc VCC bất thường) | Cả 3 cảm biến |
| Sensor timeout | Không có giá trị cập nhật mới trong khoảng thời gian quy định | Cả 3 cảm biến |
| CAN communication timeout | GUI không nhận được bản tin CAN trong khoảng thời gian quy định | Giám sát ở phía GUI, có thể phản ánh về trạng thái CAN Status trên GUI |

### 9.3. Cơ chế debounce đơn giản

Để tránh báo lỗi giả do nhiễu tức thời, DEM áp dụng bộ đếm debounce đơn giản: một lỗi chỉ được **Confirm** (chuyển thành DTC chính thức, ghi NvM) sau khi trạng thái lỗi được phát hiện liên tục trong N chu kỳ giám sát (ví dụ N = 3–5 chu kỳ). Đây là phiên bản rút gọn của debounce counter trong DEM chuẩn AUTOSAR.

---

## 10. Thiết kế DTC

Đề xuất **6 DTC** — đủ để minh họa toàn bộ các loại lỗi mà không gây quá tải quản lý.

| DTC (dạng OBD-II style) | Tên | Điều kiện | Nguồn |
|---|---|---|---|
| P0217 | Coolant Over Temperature | Coolant > ngưỡng trên | Coolant Temp |
| P0522 | Oil Pressure Low | Oil Pressure < ngưỡng dưới | Oil Pressure |
| P0562 | Battery Under Voltage | Battery Voltage < ngưỡng dưới | Battery Voltage |
| P0116 | Coolant Sensor Signal Invalid | ADC ngoài dải vật lý hợp lệ | Coolant Temp |
| P0117 | Sensor Signal Timeout | Không có dữ liệu mới trong thời gian quy định | Bất kỳ cảm biến nào (dùng chung 1 mã, phân biệt bằng dữ liệu bổ sung nếu cần) |
| U0100 | CAN Communication Timeout | GUI không nhận được bản tin ECU trong thời gian quy định | Giao tiếp CAN |

> Ghi chú: Mã DTC dùng định dạng gợi nhớ theo chuẩn OBD-II (P0xxx cho Powertrain, U0xxx cho Network) mang tính minh họa học thuật, không nhất thiết trùng khớp 100% với mã thật trong ISO 15031-6/SAE J2012 — điều này nên được nêu rõ với giảng viên (xem mục 23).

### 10.1. Cấu trúc trạng thái DTC (status byte rút gọn)

| Bit/Field | Ý nghĩa |
|---|---|
| TestFailed | Lỗi đang xảy ra ở lần kiểm tra gần nhất |
| Pending | Lỗi được phát hiện nhưng chưa đủ số chu kỳ để Confirm |
| Confirmed | Lỗi đã được xác nhận (đủ debounce), được ghi vào NvM |
| TestFailedSinceLastClear | Đã từng fail kể từ lần Clear DTC gần nhất |
| OccurrenceCounter | Số lần lỗi được Confirm |

---

## 11. Thiết kế NVM

**Kết luận: Có nên lưu DTC vào NVM — Có**, vì đây là một trong những giá trị học thuật cốt lõi của đồ án (minh họa tính bền vững của chẩn đoán qua các lần reset ECU), và mức độ phức tạp triển khai ở quy mô rút gọn là khả thi.

| Câu hỏi | Trả lời |
|---|---|
| Dữ liệu nào được lưu? | Với mỗi DTC: mã DTC, status byte (Confirmed, TestFailedSinceLastClear), OccurrenceCounter |
| Khi nào lưu? | Khi DEM Confirm một lỗi mới, hoặc khi trạng thái DTC thay đổi đáng kể (ví dụ: Clear DTC theo lệnh) — **không ghi ở mỗi chu kỳ** để tránh hao mòn bộ nhớ flash/EEPROM |
| Khi ECU reset thì đọc lại như thế nào? | Khi khởi động (Init), NvM đọc toàn bộ block DTC đã lưu, DEM khôi phục lại danh sách DTC Confirmed trước khi bắt đầu giám sát mới |
| Cần mô phỏng NVM hay dùng flash thật? | Đề xuất dùng **EEPROM nội bộ MCU** nếu MCU hỗ trợ (ví dụ STM32 có thể dùng 1 sector Flash mô phỏng EEPROM qua thư viện EEPROM emulation), hoặc dùng **EEPROM ngoài qua I2C** (ví dụ AT24C32) nếu đơn giản hơn về mặt lập trình. Nếu cả hai phương án đều phức tạp so với thời gian còn lại, phương án đơn giản hơn là dùng một vùng RAM có backup pin (nếu board hỗ trợ) — nhưng đây chỉ nên là phương án dự phòng cuối cùng vì không thể hiện đúng "non-volatile". |
| Phương án đơn giản hóa nếu NVM thật quá phức tạp | Dùng EEPROM I2C ngoài (rẻ, phổ biến, thư viện đơn giản) thay vì Flash emulation nội bộ — độ phức tạp lập trình thấp hơn đáng kể |

---

## 12. Thiết kế CAN

### 12.1. Cấu hình chung

- Baudrate: 500 kbps (chuẩn phổ biến, dễ tương thích với module CAN-USB).
- Frame format: Standard (11-bit ID), đủ dùng cho số lượng bản tin ít.

### 12.2. Danh sách bản tin CAN

| CAN ID | Tên bản tin | DLC | Chu kỳ | Chiều |
|---|---|---|---|---|
| 0x100 | Sensor Data Message | 8 | 100 ms | ECU → GUI |
| 0x200 | Diagnostic Status Message | 8 | Event-triggered (khi DTC thay đổi) + heartbeat 1000 ms | ECU → GUI |
| 0x300 | ECU Status Message | 4 | 500 ms | ECU → GUI |
| 0x400 | Fault Injection Command | 2 | Event-triggered (khi người dùng thao tác trên GUI) | GUI → ECU |
| 0x500 | OBD-II Style Request | 3 | On-demand | OBD-II Reader → ECU |
| 0x501 | OBD-II Style Response | 8 | On-demand (đáp ứng 0x500) | ECU → OBD-II Reader |

### 12.3. Data layout chi tiết

**0x100 – Sensor Data Message (8 byte)**

| Byte | Nội dung |
|---|---|
| 0–1 | Coolant Temperature (°C, scaled x10, signed) |
| 2–3 | Oil Pressure (kPa, scaled x10) |
| 4–5 | Battery Voltage (V, scaled x100) |
| 6 | Sensor Status Flags (bit0: Coolant valid, bit1: Oil valid, bit2: Battery valid) |
| 7 | Reserved |

**0x200 – Diagnostic Status Message (8 byte)**

| Byte | Nội dung |
|---|---|
| 0 | Số lượng DTC đang Confirmed |
| 1–6 | DTC bitmap hoặc danh sách tối đa 3 DTC gần nhất (2 byte/DTC) |
| 7 | Reserved |

**0x300 – ECU Status Message (4 byte)**

| Byte | Nội dung |
|---|---|
| 0 | ECU State (0: Init, 1: OK, 2: Degraded, 3: Fault) |
| 1 | CAN Tx counter (heartbeat, tăng dần mỗi chu kỳ) |
| 2–3 | Reserved |

**0x400 – Fault Injection Command (2 byte)**

| Byte | Nội dung |
|---|---|
| 0 | Fault ID (1: Coolant OverTemp, 2: Oil Low, 3: Battery UnderVoltage, 4: Sensor Invalid, 5: CAN Timeout, 0: Clear all) |
| 1 | Action (1: Inject, 0: Clear) |

**0x500/0x501 – OBD-II Style Request/Response**

Đơn giản hóa theo tinh thần Mode 03 (Read DTC) và Mode 04 (Clear DTC) của OBD-II:

| Byte (Request) | Nội dung |
|---|---|
| 0 | Mode (0x03: Read DTC, 0x04: Clear DTC) |
| 1–2 | Reserved |

| Byte (Response) | Nội dung |
|---|---|
| 0 | Mode echo |
| 1 | Số DTC |
| 2–7 | Tối đa 3 DTC (2 byte/DTC) |

### 12.4. Luồng xử lý CAN

```
Application/RTE (dữ liệu cần gửi)
   ↓
COM (đóng gói signal → PDU)
   ↓
PduR (định tuyến)
   ↓
CanIf (ánh xạ PDU ↔ CAN ID)
   ↓
CAN Driver (gửi frame vật lý)
   ↓
CAN Bus
   ↓ (chiều nhận, đối với Fault Injection Command và OBD-II Request)
CAN Driver → CanIf → PduR → COM → RTE → Application (Diagnostic SWC / Fault Injection Handler)
```

---

## 13. Thiết kế GUI

### 13.1. Nền tảng

Màn hình OLED I2C (ví dụ SSD1306, 128x64), điều khiển hiển thị bằng vi điều khiển riêng (có thể dùng chính MCU thứ 2, hoặc một MCU nhỏ khác chuyên cho GUI) kết hợp 2–3 nút bấm để điều hướng menu (Up/Down/Select), do kích thước màn hình nhỏ không thể hiển thị mọi thông tin cùng lúc.

### 13.2. Cấu trúc màn hình (menu-based, do OLED nhỏ)

| Màn hình | Nội dung hiển thị |
|---|---|
| **Sensor Dashboard** | Coolant Temp, Oil Pressure, Battery Voltage (dạng số + đơn vị), cập nhật theo bản tin 0x100 |
| **ECU Status** | Trạng thái ECU (OK/Degraded/Fault), CAN heartbeat indicator |
| **Diagnostic/DTC Display** | Danh sách DTC hiện hành (mã + tên ngắn gọn) |
| **CAN Communication Status** | Trạng thái kết nối CAN (Connected/Timeout), số bản tin nhận được |
| **Fault Injection Menu** | Danh sách lỗi có thể inject, chọn bằng nút bấm, gửi lệnh 0x400 |
| **Log đơn giản** (tùy chọn nếu còn thời gian) | Hiển thị 3–5 sự kiện gần nhất (dạng danh sách cuộn) |

### 13.3. Nguyên tắc thiết kế

GUI **chỉ hiển thị dữ liệu mà ECU chủ động gửi qua CAN** (kiến trúc thụ động/display-only đối với dữ liệu cảm biến và chẩn đoán), ngoại trừ chức năng Fault Injection là nơi GUI **chủ động gửi lệnh** xuống ECU. Điều này giữ đúng vai trò của GUI là thiết bị giám sát, không phải bộ điều khiển hay Diagnostic Tester.

---

## 14. Fault Injection

### 14.1. Cơ chế đề xuất (đơn giản, dễ demo)

Kết hợp 2 phương án bổ sung cho nhau:

1. **Qua GUI (chính, dùng cho demo)**: Người dùng chọn lỗi trong Fault Injection Menu trên OLED → gửi CAN command (0x400) → ECU nhận và giả lập trạng thái lỗi tương ứng (ví dụ ép giá trị Coolant Temp đọc được vượt ngưỡng, bỏ qua giá trị ADC thật tạm thời).
2. **Qua phần cứng (bổ sung, trực quan)**: Vặn biến trở mô phỏng cảm biến (Coolant, Oil) đến giá trị vượt ngưỡng thật — đây là cách "tự nhiên" nhất, không cần code thêm, phù hợp để demo trực quan sinh động.

### 14.2. Luồng xử lý

```
GUI (chọn Fault Injection) hoặc Biến trở vật lý (thay đổi giá trị thật)
   ↓
CAN (0x400 command) hoặc nội bộ ECU (giá trị ADC thay đổi)
   ↓
Fault Injection Handler (trong Application Layer, chèn giá trị giả hoặc set flag override)
   ↓
Sensor Monitoring SWC (đọc giá trị đã bị override hoặc giá trị thật vượt ngưỡng)
   ↓
Diagnostic SWC → DEM → DTC → NvM
   ↓
COM/PduR/CanIf/CAN Driver → CAN Bus
   ↓
GUI hiển thị lỗi
```

### 14.3. Danh sách kịch bản Fault Injection đề xuất

- Inject Coolant Over Temperature.
- Inject Oil Pressure Low.
- Inject Battery Under Voltage.
- Inject Sensor Invalid (giả lập đứt/chập cảm biến).
- Inject CAN Timeout (tạm dừng gửi CAN từ ECU để mô phỏng mất kết nối).

---

## 15. Luồng dữ liệu tổng thể (end-to-end)

```
[Cảm biến vật lý] 
   → ADC/DIO (MCAL) 
   → IoHwAb (quy đổi đơn vị) 
   → RTE 
   → Sensor Monitoring SWC (kiểm tra ngưỡng/dải/timeout) 
   → RTE 
   → Diagnostic SWC (quyết định Diagnostic Event) 
   → RTE 
   → DEM (debounce, cập nhật status, xác nhận DTC) 
   → NvM (lưu DTC nếu Confirmed) 
   → RTE 
   → COM (đóng gói signal) 
   → PduR (định tuyến) 
   → CanIf (ánh xạ CAN ID) 
   → CAN Driver (gửi frame) 
   → CAN Bus 
   → [GUI Device] nhận frame → hiển thị Sensor/ECU Status/DTC
   → [OBD-II Reader] gửi request 0x500 → ECU phản hồi 0x501 → hiển thị DTC đọc được
```

---

## 16. Use Case

| Mã | Tên Use Case | Actor | Mô tả ngắn |
|---|---|---|---|
| UC-01 | Giám sát cảm biến bình thường | ECU | ECU đọc và gửi dữ liệu cảm biến định kỳ, không có lỗi |
| UC-02 | Phát hiện lỗi cảm biến | ECU | ECU phát hiện giá trị bất thường, tạo DTC, gửi CAN |
| UC-03 | Lưu và khôi phục DTC sau reset | ECU | DTC được lưu NvM, khôi phục sau khi ECU khởi động lại |
| UC-04 | Hiển thị dữ liệu trên GUI | Người dùng, GUI | Người vận hành xem dữ liệu cảm biến/DTC trên OLED |
| UC-05 | Tạo lỗi giả lập (Fault Injection) | Người dùng, GUI | Người vận hành chủ động tạo tình huống lỗi để demo |
| UC-06 | Đọc DTC qua OBD-II Reader | Người dùng, OBD-II Reader | Người vận hành dùng thiết bị đọc riêng để lấy danh sách DTC |
| UC-07 | Xóa DTC (Clear DTC) | Người dùng, GUI hoặc OBD-II Reader | Xóa toàn bộ DTC đã lưu, reset trạng thái chẩn đoán |
| UC-08 | Phát hiện mất kết nối CAN | GUI | GUI phát hiện không nhận được bản tin ECU trong thời gian quy định, hiển thị cảnh báo |

---

## 17. Test Plan

| Hạng mục | Mục tiêu kiểm thử | Phương pháp |
|---|---|---|
| Sensor acquisition | ADC/DIO đọc đúng giá trị, quy đổi đơn vị chính xác | So sánh giá trị đọc được với đồng hồ đo tham chiếu |
| Sensor monitoring | Phát hiện đúng các trạng thái Normal/Over/Under/Invalid/Timeout | Test với giá trị input được kiểm soát (biến trở, mô phỏng) |
| Fault detection | Debounce hoạt động đúng, không báo lỗi giả với nhiễu ngắn | Tạo xung nhiễu ngắn hạn, kiểm tra không Confirm DTC |
| DTC creation | DTC được tạo đúng mã, đúng điều kiện | Kiểm tra bảng DTC sau khi Inject từng loại lỗi |
| DTC status | Status byte cập nhật đúng (Pending → Confirmed), OccurrenceCounter tăng đúng | Kiểm tra qua log debug hoặc CAN message |
| NVM persistence | DTC còn tồn tại sau khi reset ECU | Tạo lỗi → reset ECU → kiểm tra DTC còn nguyên |
| CAN communication | Đúng CAN ID, đúng chu kỳ, đúng data layout | Dùng CAN analyzer / logger để bắt và so sánh |
| GUI | Hiển thị đúng dữ liệu nhận được, cập nhật kịp thời | So sánh dữ liệu trên GUI với dữ liệu gửi từ ECU |
| Fault Injection | Lệnh inject từ GUI/nút bấm tạo đúng lỗi tương ứng | Test từng loại lỗi trong danh sách mục 14.3 |
| OBD-II Reader | Đọc đúng danh sách DTC theo yêu cầu, Clear DTC hoạt động | Gửi request 0x500, kiểm tra response 0x501 |

### 17.1. Test case mẫu (end-to-end)

```
Test Case: Coolant Over Temperature

Bước 1: Coolant = 120 °C (vượt ngưỡng 110 °C)
Bước 2: Sensor Monitoring SWC phát hiện OverRange
Bước 3: Diagnostic SWC tạo Diagnostic Event
Bước 4: DEM debounce đủ chu kỳ → Confirm
Bước 5: DTC P0217 được tạo, lưu vào NvM
Bước 6: COM/PduR/CanIf/CAN Driver gửi bản tin 0x200
Bước 7: GUI nhận bản tin, hiển thị:
        "P0217 - COOLANT OVER TEMPERATURE"

Kết quả mong đợi: GUI hiển thị đúng mã và tên lỗi trong vòng ≤ 1.5s kể từ khi Coolant vượt ngưỡng đủ số chu kỳ debounce.
```

---

## 18. Phân chia công việc 2 người

Cách chia ban đầu theo đề xuất là hợp lý, giữ nguyên với một vài điều chỉnh nhỏ để cân bằng khối lượng công việc (Thành viên 2 phụ trách CAN + GUI + Fault Injection có khối lượng khá lớn, nên bổ sung DIO/OBD-II Reader vào phía Thành viên 2 và để Thành viên 1 hỗ trợ IoHwAb/OBD-II export ở phía ECU).

### Thành viên 1 – "ECU Core & Diagnosis"

- MCU setup, cấu hình project.
- Sensor (Coolant, Oil, Battery) + mạch mô phỏng (biến trở, chia áp).
- ADC/DIO Driver (MCAL).
- IoHwAb.
- RTE (thiết kế và viết tay các hàm Rte_Read/Write/Call).
- Sensor Monitoring SWC.
- Diagnostic SWC.
- DEM (debounce, status byte, DTC table).
- NvM (lưu/đọc DTC).
- OBD-II style export logic (phía ECU, xử lý request 0x500/0x501).

### Thành viên 2 – "Communication & GUI"

- CAN Driver bọc lại (MCAL wrapper).
- CanIf, PduR, COM (thiết kế PDU, signal mapping).
- Thiết kế bản tin CAN (CAN ID, DLC, data layout — phối hợp với Thành viên 1).
- GUI trên OLED I2C (thiết kế menu, vẽ giao diện, xử lý nút bấm).
- Fault Injection (giao diện chọn lỗi trên GUI + gửi lệnh CAN).
- OBD-II Reader (thiết bị/chương trình đọc riêng).
- CAN monitoring/logging phục vụ test và debug.
- Tích hợp hệ thống (Integration) — phối hợp cùng Thành viên 1 ở Phase 8.

### Công việc chung (cả 2 người)

- Thiết kế kiến trúc tổng thể (Phase 1).
- Test end-to-end và chuẩn bị demo (Phase 9).
- Viết báo cáo đồ án.

---

## 19. Kế hoạch 12 tuần

| Tuần | Phase | Công việc | Người phụ trách | Deliverable | Milestone |
|---|---|---|---|---|---|
| 1 | Phase 1 – Architecture | Thiết kế kiến trúc tổng thể, phân lớp AUTOSAR rút gọn, chọn MCU/module CAN/OLED | Cả 2 | Tài liệu kiến trúc (bản vẽ + mô tả module) | ✅ Kiến trúc được duyệt |
| 2 | Phase 1 – Architecture | Thiết kế chi tiết interface giữa các lớp (RTE API, DEM API, COM signal list), chọn cảm biến + mạch mô phỏng | Cả 2 | Bảng interface, sơ đồ CAN ID sơ bộ | Interface spec hoàn chỉnh |
| 3 | Phase 2 – Sensor + MCAL + IoHwAb | Setup ADC/DIO driver, đọc thử cảm biến, mạch mô phỏng biến trở | TV1 | ADC/DIO driver hoạt động, đọc giá trị thô | Đọc được 3 cảm biến |
| 4 | Phase 2 – Sensor + MCAL + IoHwAb | Hoàn thiện IoHwAb (quy đổi đơn vị), song song: TV2 setup CAN Driver cơ bản | TV1 (IoHwAb), TV2 (CAN Driver) | IoHwAb module, CAN loopback test | Giá trị cảm biến quy đổi đúng đơn vị |
| 5 | Phase 3 – RTE + Application SWC | Viết RTE rút gọn, khung Sensor Monitoring SWC | TV1 | RTE + Sensor Monitoring SWC (chưa có threshold logic) | Dữ liệu chảy từ Sensor đến SWC qua RTE |
| 6 | Phase 3 – RTE + Application SWC | Hoàn thiện threshold/range/timeout logic trong Sensor Monitoring SWC | TV1 | Sensor Monitoring SWC hoàn chỉnh | Phát hiện đúng Normal/Over/Under/Invalid/Timeout |
| 7 | Phase 4 – Diagnosis + DEM + DTC | Diagnostic SWC + DEM (debounce, status byte, bảng 6 DTC) | TV1 | DEM + DTC table hoạt động | Tạo đúng DTC khi có lỗi |
| 8 | Phase 5 – CAN | Hoàn thiện CanIf/PduR/COM, định nghĩa đầy đủ bản tin CAN (0x100–0x501) | TV2 | CAN communication hoạt động (ECU gửi được dữ liệu) | Bắt được đúng bản tin bằng CAN analyzer |
| 9 | Phase 6 – GUI | Thiết kế và code GUI trên OLED (Sensor Dashboard, ECU Status, DTC Display) | TV2 | GUI hiển thị dữ liệu nhận từ CAN | GUI hiển thị đúng dữ liệu thời gian thực |
| 10 | Phase 6 – GUI + Phase 7 – NVM | Hoàn thiện GUI (CAN Status, Fault Injection Menu); song song TV1 làm NvM | TV2 (GUI), TV1 (NvM) | GUI hoàn chỉnh, NvM lưu/đọc DTC | DTC còn sau khi reset ECU |
| 11 | Phase 8 – Integration | Tích hợp toàn bộ hệ thống, Fault Injection end-to-end, OBD-II Reader | Cả 2 | Hệ thống tích hợp đầy đủ | Luồng end-to-end chạy được từ Sensor đến GUI |
| 12 | Phase 9 – Testing + Demo | Chạy Test Plan (mục 17), sửa lỗi, tập kịch bản demo, viết báo cáo | Cả 2 | Test report, kịch bản demo hoàn chỉnh, báo cáo đồ án | ✅ Sẵn sàng bảo vệ đồ án |

---

## 20. Kịch bản demo (5–10 phút)

1. **Khởi động ECU** — bật nguồn, quan sát GUI hiển thị trạng thái "Init → OK".
2. **Sensor hoạt động bình thường** — GUI hiển thị Coolant/Oil/Battery ở giá trị bình thường, không có DTC.
3. **GUI hiển thị dữ liệu thời gian thực** — chuyển qua các màn hình Sensor Dashboard / ECU Status để minh họa.
4. **Tạo lỗi cảm biến** — vặn biến trở Coolant vượt ngưỡng (hoặc chọn Fault Injection trên GUI).
5. **ECU phát hiện lỗi** — sau vài trăm ms (đủ debounce), trạng thái chuyển sang Fault.
6. **DEM tạo DTC** — DTC P0217 được Confirm.
7. **DTC được lưu vào NVM** — (có thể minh họa bằng cách reset ECU ngay sau đó và cho thấy DTC vẫn còn).
8. **ECU gửi trạng thái qua CAN** — dùng CAN logger để show trực tiếp bản tin 0x200 trên màn hình laptop (tùy chọn, tăng tính thuyết phục).
9. **GUI hiển thị lỗi** — chuyển sang màn hình DTC Display, hiển thị "P0217 - COOLANT OVER TEMPERATURE".
10. **Khôi phục sensor** — vặn biến trở về giá trị bình thường.
11. **ECU cập nhật trạng thái** — TestFailed = false nhưng TestFailedSinceLastClear vẫn true (DTC vẫn tồn tại cho đến khi Clear).
12. **Clear DTC** (nếu đã triển khai) — dùng OBD-II Reader gửi lệnh Clear (Mode 04 style), quan sát GUI/OBD-II Reader không còn DTC.

---

## 21. Các rủi ro

| Rủi ro | Mức độ | Biện pháp giảm thiểu |
|---|---|---|
| Tích hợp CAN giữa 2 module (ECU/GUI) gặp lỗi phần cứng (transceiver, dây nối, ground loop) | Cao | Test CAN loopback sớm (Tuần 4), dùng CAN analyzer để debug từ đầu |
| NvM/EEPROM phức tạp hơn dự kiến, tốn nhiều thời gian debug | Trung bình | Có phương án dự phòng đơn giản hơn (mục 11); chốt sớm phương án ở Tuần 2 |
| Timeline bị trễ do 1 trong 2 người gặp khó khăn kỹ thuật ở phần riêng | Trung bình | Có buổi sync hàng tuần, chia nhỏ deliverable theo tuần để phát hiện trễ sớm |
| Phạm vi bị mở rộng quá đà (feature creep) do muốn thêm UDS/Diagnostic Tester đầy đủ | Cao nếu không kiểm soát | Chốt scope với giảng viên từ đầu (mục 23), review scope định kỳ mỗi Phase |
| Debounce/threshold không phù hợp gây báo lỗi giả hoặc bỏ sót lỗi khi demo | Trung bình | Tinh chỉnh ngưỡng qua thử nghiệm thực tế ở Phase 9, để thời gian buffer cho hiệu chỉnh |
| Màn hình OLED nhỏ khó hiển thị đủ thông tin, giao diện menu phức tạp hơn dự kiến | Thấp–Trung bình | Thiết kế menu tối giản ngay từ đầu (mục 13.2), ưu tiên rõ ràng hơn là đầy đủ |

---

## 22. Các chức năng Optional/Future Work

Các mục sau **không** thuộc core scope, chỉ nên thực hiện nếu còn dư thời gian đáng kể sau Phase 9, và **không được đánh đổi bằng việc làm sơ sài core project**:

| Chức năng | Lý do không cần thiết cho core project |
|---|---|
| **Diagnostic Tester riêng theo UDS đầy đủ (ISO 14229)** | Yêu cầu triển khai Session Control, Security Access, Routine Control... vượt xa mức cần thiết để minh họa tư duy AUTOSAR + fault diagnosis cơ bản; tốn thời gian không tương xứng với 3 tháng/2 người |
| **Gateway ECU** | Đồ án chỉ có 1 mạng CAN, không có nhu cầu định tuyến liên mạng, thêm Gateway không phục vụ mục tiêu giám sát cảm biến |
| **Bootloader / ECU reprogramming** | Là một lĩnh vực kỹ thuật riêng biệt (flash qua CAN, security), không liên quan trực tiếp đến giám sát cảm biến và chẩn đoán lỗi |
| **Triển khai đầy đủ AUTOSAR Classic (BSW đầy đủ, RTE Generator, ARXML)** | Cần công cụ thương mại (Vector DaVinci, EB tresos...) và khối lượng cấu hình rất lớn, không khả thi cho đồ án sinh viên trong 3 tháng |
| **Thêm cảm biến Fuel Level, Door Status** | Không làm tăng độ khó kỹ thuật cốt lõi nhưng tăng thời gian tích hợp/hiệu chỉnh; có thể bổ sung Door Status (DIO) nếu dư thời gian vì tận dụng được DIO Driver đã có |
| **Log dữ liệu lên SD Card / máy tính** | Hữu ích cho phân tích sau demo nhưng không phải yêu cầu lõi để chứng minh luồng Sensor→ECU→Diagnosis→DTC→CAN→GUI |
| **Giao diện GUI trên PC (thay vì chỉ OLED)** | Có thể làm phong phú demo (dashboard đẹp hơn) nhưng OLED đã đủ để chứng minh chức năng; PC GUI có thể là hướng mở rộng báo cáo |
| **Security (mã hóa CAN, xác thực)** | Ngoài phạm vi một hệ thống giám sát/chẩn đoán học thuật, thuộc lĩnh vực Automotive Cybersecurity riêng biệt |

---

## 23. Scope cuối cùng nên chốt với giảng viên

Trước khi bắt đầu triển khai (cuối Tuần 1), nhóm nên trình bày và xin xác nhận từ giảng viên hướng dẫn về các điểm sau, để tránh tranh cãi về phạm vi ở giai đoạn cuối:

1. **AUTOSAR "theo tư duy" chứ không phải AUTOSAR chuẩn đầy đủ** — không dùng công cụ cấu hình thương mại, RTE và các module Service Layer được viết tay bằng C theo đúng vai trò/interface nhưng ở mức rút gọn.
2. **Số lượng cảm biến: 3** (Coolant Temp, Oil Pressure, Battery Voltage) + OLED GUI — không bắt buộc mở rộng thêm.
3. **Số lượng DTC: 6** — đủ minh họa các loại lỗi (over-range, under-range, invalid, timeout, CAN timeout), không cần nhiều hơn.
4. **OBD-II chỉ ở mức "phong cách" (OBD-II style)** — mô phỏng tinh thần Mode 03/04 (đọc/xóa DTC) qua CAN tự định nghĩa, **không phải triển khai đầy đủ ISO 15031/SAE J1979** với đầy đủ PID và giao thức chuẩn.
5. **GUI là thiết bị giám sát (monitoring display)**, không phải Diagnostic Tester theo chuẩn UDS — chức năng Fault Injection và đọc DTC là chức năng phụ trợ phục vụ demo, không đại diện cho một Diagnostic Tester công nghiệp.
6. **NVM dùng EEPROM (nội bộ hoặc I2C ngoài)** ở mức đơn giản hóa, không cần cơ chế wear-leveling hay redundancy phức tạp.
7. **UDS đầy đủ, Gateway ECU, Bootloader/reprogramming, toàn bộ AUTOSAR BSW** đều nằm ngoài phạm vi, chỉ được đề cập như Future Work trong báo cáo, không phải yêu cầu chấm điểm.
8. **Tiêu chí thành công của đồ án** là chứng minh được luồng end-to-end hoạt động ổn định: Sensor → ECU (theo tư duy AUTOSAR) → Fault Diagnosis → DTC → CAN → GUI → OBD-II Reader, kèm theo tài liệu kiến trúc và test report rõ ràng — không phải quy mô hay số lượng tính năng.

---

*Tài liệu này là đề cương tổng thể cho đồ án tốt nghiệp, có thể được điều chỉnh nhỏ trong quá trình triển khai thực tế, miễn là không làm thay đổi phạm vi cốt lõi đã chốt ở mục 23.*
