## 1. Kiến trúc tổng quan: Host vs Controller

BLE stack chia làm 2 lớp tách biệt, giao tiếp với nhau qua **HCI (Host Controller Interface)**:

```
 ┌───────────────────────────────────────┐
 │            APPLICATION                │   ← code của bạn
 └──────────────────┬────────────────────┘
                    │  GAP/GATT API
 ┌──────────────────▼────────────────────┐
 │              HOST                     │   ← NimBLE / Bluedroid
 │  (GAP, GATT, SMP, L2CAP, ATT...)      │      chạy trên CPU chính
 └──────────────────┬────────────────────┘
                    │  HCI (UART / SPI / trực tiếp trong chip)
 ┌──────────────────▼────────────────────┐
 │            CONTROLLER                 │   ← lõi radio BLE
 │  (Link Layer, PHY, timing chuẩn xác)  │      xử lý sóng vô tuyến
 └──────────────────┬────────────────────┘
                    │
                antenna
```

- **Controller**: phần cứng radio, xử lý timing cực chính xác (advertising interval, connection interval...). Trên ESP32, đây là lõi riêng chạy firmware Bluetooth độc lập.
- **Host**: xử lý logic — quản lý kết nối, service/characteristic (GATT), bảo mật (SMP)...
- **HCI**: giao thức chuẩn để 2 bên nói chuyện với nhau, gồm Commands (host→controller), Events (controller→host), và ACL Data (dữ liệu ứng dụng).

Đây là lý do bước **sync** cần thiết mà mình nói ở câu trước — host phải bắt tay HCI với controller trước khi tin dùng được nó.

## 2. Vòng đời một thiết bị BLE (từ góc nhìn app)

```
   [Power on]
       │
       ▼
 ble_hs reset controller (HCI_Reset)
       │
       ▼
 host đọc capability của controller
 (BD_ADDR, buffer size, supported features...)
       │
       ▼
   sync_cb() được gọi  ◄── "sẵn sàng dùng BLE"
       │
       ▼
 ┌─────────────────────────────┐
 │   app chọn 1 trong các vai  │
 └─────────────────────────────┘
       │
   ┌───┴────────┬───────────┬───────────┐
   ▼            ▼           ▼           ▼
Advertiser   Scanner     Central    Peripheral
(quảng bá)   (dò tìm     (chủ động  (bị động
             thiết bị    kết nối)   chờ kết nối)
             khác)
```

## 3. Hai vai trò chính (GAP roles)

**Peripheral (ví dụ: cảm biến, thiết bị IoT nhỏ):**
```
 advertise() ──► [đợi] ──► central kết nối tới ──► connected
     │
     └─ gói tin quảng bá: tên thiết bị, service UUID...
```

**Central (ví dụ: điện thoại, gateway):**
```
 scan() ──► thấy advertising packet ──► connect() ──► connected
```

Sau khi **connected**, cả 2 bên đều có thể dùng GATT để trao đổi dữ liệu.

## 4. Lớp GATT — nơi thực sự trao đổi dữ liệu

```
 Peripheral (GATT Server)          Central (GATT Client)
 ┌───────────────────────┐
 │ Service (UUID)        │
 │  ├─ Characteristic 1  │  ◄──── read/write/notify ────  App đọc/ghi
 │  │    └─ Value        │
 │  ├─ Characteristic 2  │
 │  │    └─ Descriptor   │
 └───────────────────────┘
```

- **Server** thường là thiết bị Peripheral (giữ dữ liệu, ví dụ cảm biến nhiệt độ).
- **Client** thường là Central (đọc dữ liệu đó).
- Characteristic có thể **notify** — server chủ động đẩy dữ liệu mới cho client mà không cần client hỏi lại (dùng nhiều trong IoT để đẩy sensor data liên tục).

## 5. Khớp lại với 2 callback đã hỏi trước

```
 startup
    │
    ▼
 [HCI handshake] ── nếu lỗi controller nghiêm trọng ──► reset_cb()
    │                                                       │
    ▼                                                       │
 sync_cb() ◄─────────────── host tự re-sync ────────────────┘
    │
    ▼
 app bắt đầu: ble_app_advertise() hoặc scan()
```

`reset_cb` và `sync_cb` chỉ nằm ở **lớp dưới cùng** (host↔controller) — chúng đảm bảo nền tảng ổn định trước khi bạn động tới GAP/GATT ở phía trên.