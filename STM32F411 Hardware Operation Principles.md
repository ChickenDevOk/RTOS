# Nguyên lý hoạt động phần cứng STM32F411

## Mục lục
1. Kiến trúc hệ thống
2. Hệ thống Clock
3. GPIO
4. Timer System
5. ADC
6. UART/USART
7. I2C
8. SPI
9. DMA
10. Interrupt System

---

## 1. Kiến trúc hệ thống

### Kiến trúc ARM Cortex-M4
- **Đơn vị xử lý trung tâm**
  - Kiến trúc Harvard: Tách biệt bus dữ liệu và bus lệnh
  - Pipeline 3 tầng: Fetch → Decode → Execute
  - Hỗ trợ DSP và FPU (Floating Point Unit)

### Bus Matrix
- **AHB (Advanced High-performance Bus)**
  - Tốc độ cao (lên đến 100MHz)
  - Dùng cho các thiết bị ngoại vi quan trọng
  - Kết nối CPU, DMA, Flash memory

- **APB (Advanced Peripheral Bus)**
  - Tốc độ thấp hơn
  - Dùng cho các thiết bị ngoại vi thông thường
  - Tiết kiệm năng lượng

---

## 2. Hệ thống Clock

### Nguồn clock
1. **HSI (High-Speed Internal)**
  - Dao động RC nội 16MHz
  - Độ chính xác ±1%
  - Khởi động nhanh

2. **HSE (High-Speed External)**
  - Crystal/Ceramic resonator 4-26MHz
  - Độ chính xác cao
  - Thời gian khởi động lâu hơn

3. **PLL (Phase-Locked Loop)**
  - Nhân tần số từ HSI/HSE
  - Tạo ra tần số cao hơn (lên đến 100MHz)
  - Công thức: \[f_{OUT} = f_{IN} × \frac{N}{M × P}\]

### Clock tree
```
HSI (16MHz) ─┐
             ├─→ PLL →─┐
HSE (8MHz) ─┘         │
                      ├─→ System Clock (max 100MHz)
                      │
LSI (32KHz) ─────────┤
                      │
LSE (32.768KHz) ────┘
```

---

## 3. GPIO (General Purpose Input/Output)

### Cấu trúc một chân GPIO
```
                   ┌─────────┐
Input path     ────┤  Schmitt│
                   │ Trigger │
                   └────┬────┘
                        │
                   ┌────┴────┐
                   │  Input  │
                   │Register │
                   └────┬────┘
                        │
Alternative     ────────┤
Function            ┌───┴───┐
                   │ Output │
                   │Register│
                   └───┬───┘
                       │
                  ┌───┴───┐
                  │ Driver│
                  └───┬───┘
                      │
Pin              ────┴────
```

### Mode hoạt động
1. **Input mode**
  - Floating input
  - Pull-up input
  - Pull-down input
  
2. **Output mode**
  - Push-pull output
  - Open-drain output
  
3. **Alternate Function**
  - 16 chức năng thay thế cho mỗi pin
  - Kết nối với các peripheral bên trong
  
4. **Analog mode**
  - Bypass digital input/output
  - Kết nối trực tiếp với ADC

---

## 4. Timer System

### Cấu trúc cơ bản của Timer
```
              Clock          Compare
                │              │
                ▼              ▼
┌──────────┐  ┌─────┐  ┌──────────┐
│Prescaler ├─►│Count├─►│ Compare  ├─► Output
└──────────┘  └─────┘  └──────────┘
                 │
                 ▼
             Overflow
```

### Nguyên lý hoạt động
1. **Prescaler**
  - Chia tần số đầu vào
  - \[f_{timer} = \frac{f_{clock}}{prescaler + 1}\]

2. **Counter**
  - Đếm theo xung clock đã chia
  - Auto-reload khi đạt giá trị ARR
  - Các mode đếm:
    * Đếm lên
    * Đếm xuống
    * Đếm lên xuống (Center-aligned)

3. **Output Compare**
  - So sánh giá trị đếm với giá trị cài đặt
  - Tạo ra các tín hiệu PWM
  - \[Duty Cycle = \frac{CCR}{ARR} × 100\%\]

---

## 5. ADC (Analog-to-Digital Converter)

### Cấu trúc ADC
```
Analog Input ─► Sample/Hold ─► ADC Core ─► Digital Output
                    │             │
                    ▼             ▼
                 Capacitor    Successive
                             Approximation
```

### Nguyên lý hoạt động
1. **Sample and Hold**
  - Lấy mẫu điện áp analog
  - Giữ điện áp ổn định trong quá trình chuyển đổi
  - Thời gian lấy mẫu: \[t_{sample} = (R_{source} + R_{internal}) × C_{sh}\]

2. **Successive Approximation**
  - So sánh từng bit, từ MSB đến LSB
  - 12 chu kỳ clock cho độ phân giải 12-bit
  - Công thức chuyển đổi:
    \[Digital\_Value = \frac{V_{in} × 4096}{V_{ref}}\]

3. **Chế độ chuyển đổi**
  - Single conversion
  - Continuous conversion
  - Scan mode
  - Discontinuous mode

---

## 6. UART/USART

### Cấu trúc UART
```
         TX Data                        RX Data
           │                              ▲
           ▼                              │
┌────────┐     ┌─────┐             ┌─────┐
│Shift   ├────►│Start│─── UART ────│Stop │
│Register│     │ Bit │    Line     │ Bit │
└────────┘     └─────┘             └─────┘
```

### Nguyên lý truyền dữ liệu
1. **Idle state (Line = 1)**
2. **Start bit (Line = 0)**
3. **Data bits (LSB first)**
4. **Parity bit (optional)**
5. **Stop bit(s) (Line = 1)**

### Tính toán Baud Rate
- \[BRR = \frac{f_{CK}}{16 × BaudRate}\]
- Ví dụ: 9600 baud @ 16MHz
  \[BRR = \frac{16,000,000}{16 × 9600} = 104.166\]

---

## 7. I2C (Inter-Integrated Circuit)

### Cấu trúc bus I2C
```
VDD
 │
 ┣━━ Pull-up        Pull-up ━━┫
 │                            │
SDA ────────────────────────SDA
SCL ────────────────────────SCL
 │                            │
Master                      Slave
```

### Nguyên lý hoạt động
1. **Start Condition**
  - SDA chuyển từ HIGH → LOW khi SCL = HIGH
  
2. **Address Phase**
  - 7-bit địa chỉ + 1 bit R/W
  - Slave ACK bằng cách kéo SDA xuống LOW
  
3. **Data Phase**
  - 8-bit data + 1 bit ACK
  - MSB first
  
4. **Stop Condition**
  - SDA chuyển từ LOW → HIGH khi SCL = HIGH

### Timing Diagram
```
SDA ──┐      ┌────┐     ┌────┐     ┌──
      └──────┘    └─────┘    └─────┘
SCL ────┐   ┌─┐   ┌─┐   ┌─┐   ┌─┐
        └───┘ └───┘ └───┘ └───┘ └──
      START  bit7  bit6  bit5  bit4
```

---

## 8. SPI (Serial Peripheral Interface)

### Cấu trúc SPI
```
   Master                 Slave
┌─────────┐           ┌─────────┐
│    MOSI ├──────────►│ MOSI    │
│         │           │         │
│    MISO │◄─────────┤ MISO    │
│         │           │         │
│     SCK ├──────────►│ SCK     │
│         │           │         │
│     NSS ├──────────►│ NSS     │
└─────────┘           └─────────┘
```

### Nguyên lý hoạt động
1. **Full-duplex transmission**
  - Truyền và nhận đồng thời
  - Shift register 8/16-bit
  
2. **Clock Polarity (CPOL)**
  - CPOL=0: SCK idle ở mức thấp
  - CPOL=1: SCK idle ở mức cao
  
3. **Clock Phase (CPHA)**
  - CPHA=0: Lấy mẫu tại cạnh đầu
  - CPHA=1: Lấy mẫu tại cạnh sau

### Mode hoạt động
```
Mode  CPOL  CPHA   Clock Idle   Data Sample
 0     0     0       Low        Rising Edge
 1     0     1       Low        Falling Edge
 2     1     0       High       Falling Edge
 3     1     1       High       Rising Edge
```

---

## 9. DMA (Direct Memory Access)

### Cấu trúc DMA
```
┌─────────┐     ┌─────────┐     ┌─────────┐
│ Source  │     │   DMA   │     │  Dest   │
│ Memory  ├────►│Controller├────►│ Memory  │
└─────────┘     └─────────┘     └─────────┘
                     ▲
                     │
                 CPU request
```

### Nguyên lý hoạt động
1. **Khởi tạo**
  - Cấu hình địa chỉ nguồn và đích
  - Số lượng dữ liệu cần truyền
  - Kích thước dữ liệu (byte/half-word/word)

2. **Truyền dữ liệu**
  - DMA yêu cầu quyền truy cập bus
  - CPU tạm dừng truy cập bus
  - DMA thực hiện truyền dữ liệu
  - Giảm bộ đếm sau mỗi lần truyền

3. **Mode truyền**
  - Memory to Memory
  - Memory to Peripheral
  - Peripheral to Memory
  - Peripheral to Peripheral

---

## 10. Interrupt System

### Cấu trúc NVIC (Nested Vectored Interrupt Controller)
```
┌────────┐    ┌──────┐    ┌─────────┐
│IRQ     ├───►│ NVIC ├───►│ Cortex  │
│Sources │    │      │    │   M4    │
└────────┘    └──────┘    └─────────┘
                  │
            Priority control
```

### Nguyên lý hoạt động
1. **Interrupt Request**
  - Nguồn ngắt tạo tín hiệu IRQ
  - NVIC nhận và kiểm tra độ ưu tiên

2. **Priority Handling**
  - 16 mức độ ưu tiên (0-15)
  - Mức 0 là cao nhất
  - Ngắt có độ ưu tiên cao hơn sẽ ngắt ngắt có độ ưu tiên thấp

3. **Context Switching**
  - Lưu trạng thái hiện tại (Push registers)
  - Thực thi ISR (Interrupt Service Routine)
  - Khôi phục trạng thái (Pop registers)

### Vector Table
```
Address     Handler
0x0000      Stack pointer
0x0004      Reset
0x0008      NMI
0x000C      HardFault
...         ...
0x0040      IRQ0
0x0044      IRQ1
```

---

## Tổng kết

### Tương tác giữa các phần cứng
```
                 ┌─────────┐
                 │   CPU   │
                 └────┬────┘
                      │
          ┌──────────┴──────────┐
          │      Bus Matrix     │
          └──────────┬──────────┘
                    │
   ┌────────┬───────┴───────┬────────┐
   │        │               │        │
┌──┴──┐  ┌──┴──┐        ┌──┴──┐  ┌──┴──┐
│ DMA │  │Timer│        │ ADC │  │UART │
└─────┘  └─────┘        └─────┘  └─────┘
```

### Các điểm quan trọng
1. Tất cả các peripheral đều kết nối