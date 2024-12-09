# Bootloader
## Hiểu về Bootloader và Cách Triển khai

## Mục lục
1. Bootloader là gì?
2. Kiến trúc Bootloader
3. Memory Layout
4. Boot Process
5. Bootloader Protocol
6. Security Features
7. Triển khai trên STM32
8. Ví dụ thực tế

---

## 1. Bootloader là gì?

### Định nghĩa
- Chương trình được thực thi đầu tiên khi thiết bị khởi động
- Chịu trách nhiệm khởi tạo hệ thống
- Cho phép cập nhật firmware (In-System Programming)

### Vai trò chính
1. **Khởi tạo phần cứng**
   - Cấu hình clock
   - Khởi tạo bộ nhớ
   - Kiểm tra hệ thống

2. **Quản lý firmware**
   - Kiểm tra tính toàn vẹn
   - Cập nhật firmware
   - Rollback khi lỗi

---

## 2. Kiến trúc Bootloader

### Cấu trúc tổng quan
```
┌─────────────────┐
│   Application   │
├─────────────────┤
│   Bootloader    │
├─────────────────┤
│  Vector Table   │
└─────────────────┘
```

### Các thành phần chính
1. **Boot ROM**
   - Mã nguồn cố định
   - Không thể thay đổi
   - Chứa các hàm cơ bản

2. **Main Bootloader**
   - Có thể cập nhật
   - Xử lý chính
   - Giao thức truyền thông

3. **Application Interface**
   - Jump to application
   - Shared functions
   - System calls

---

## 3. Memory Layout

### Flash Memory Map
```
0x0800 0000 ┌─────────────────┐
            │   Bootloader    │
            │    (32KB)       │
0x0800 8000 ├─────────────────┤
            │   Application   │
            │    (448KB)      │
0x0808 0000 ├─────────────────┤
            │   Parameters    │
            │    (32KB)       │
0x0808 8000 └─────────────────┘
```

### RAM Memory Map
```
0x2000 0000 ┌─────────────────┐
            │   Bootloader    │
            │     RAM         │
0x2000 2000 ├─────────────────┤
            │   Application   │
            │     RAM         │
0x2002 0000 └─────────────────┘
```

---

## 4. Boot Process

### Quy trình khởi động
```
Power On
   │
   ▼
Reset Handler
   │
   ▼
Hardware Init
   │
   ▼
Check Boot Mode
   │
┌──┴──┐
│     │
▼     ▼
Boot   Normal
Mode   Mode
```

### Boot Mode Selection
1. **Hardware Trigger**
   - GPIO Pin
   - DIP Switch
   - Button Press

2. **Software Trigger**
   - Magic Number
   - Flag in Flash
   - CRC Error

---

## 5. Bootloader Protocol

### Command Structure
```
┌────────┬────────┬──────────┬─────┐
│ Start  │ CMD ID │ Payload  │ CRC │
│ (1B)   │ (1B)   │ (nB)    │ (2B)│
└────────┴────────┴──────────┴─────┘
```

### Basic Commands
```c
// Command IDs
#define CMD_PING          0x01
#define CMD_GET_VERSION   0x02
#define CMD_ERASE        0x03
#define CMD_WRITE        0x04
#define CMD_READ         0x05
#define CMD_VERIFY       0x06
#define CMD_RESET        0x07
```

### Protocol Flow
```
Host                  Device
 │        PING         │
 ├──────────────────► │
 │       ACK          │
 │ ◄─────────────────┤
 │     GET_VERSION    │
 ├──────────────────► │
 │    VERSION_INFO    │
 │ ◄─────────────────┤
 │      ERASE        │
 ├──────────────────► │
 │       ACK          │
 │ ◄─────────────────┤
 │      WRITE        │
 ├──────────────────► │
 │       ACK          │
 │ ◄─────────────────┤
```

---

## 6. Security Features

### Bảo mật Bootloader
1. **Mã hóa Firmware**
   - AES-256
   - RSA Public Key
   - Secure Boot

2. **Xác thực**
   ```
   ┌──────────┬──────────┬─────────┐
   │ Firmware │ Signature│ Metadata│
   └──────────┴──────────┴─────────┘
   ```

3. **Write Protection**
   - Read/Write Protection
   - Sector Protection
   - Debug Protection

---

## 7. Triển khai trên STM32

### Vector Table Relocation
```c
// Trong bootloader
SCB->VTOR = FLASH_BASE;

// Trong application
SCB->VTOR = APP_ADDRESS;
```

### Jump to Application
```c
typedef void (*pFunction)(void);

void JumpToApplication(uint32_t addr) {
    pFunction Jump_To_Application;
    uint32_t JumpAddress;
    
    // Lấy địa chỉ Reset_Handler
    JumpAddress = *(__IO uint32_t*) (addr + 4);
    Jump_To_Application = (pFunction) JumpAddress;
    
    // Cấu hình MSP
    __set_MSP(*(__IO uint32_t*) addr);
    
    // Jump to app
    Jump_To_Application();
}
```

### Flash Programming
```c
HAL_StatusTypeDef ProgramPage(uint32_t addr, uint8_t *data) {
    HAL_FLASH_Unlock();
    
    for(uint32_t i = 0; i < PAGE_SIZE; i += 4) {
        HAL_FLASH_Program(FLASH_TYPEPROGRAM_WORD,
                        addr + i,
                        *(uint32_t*)(data + i));
    }
    
    HAL_FLASH_Lock();
    return HAL_OK;
}
```

---

## 8. Ví dụ thực tế

### Simple Bootloader
```c
int main(void) {
    // 1. Khởi tạo hệ thống
    SystemInit();
    
    // 2. Kiểm tra boot mode
    if(CheckBootMode()) {
        // 3. Vào chế độ bootloader
        BootloaderMode();
    } else {
        // 4. Jump to application
        JumpToApplication(APP_ADDRESS);
    }
    
    while(1);
}
```

### UART Bootloader
```c
void BootloaderMode(void) {
    uint8_t cmd;
    
    while(1) {
        if(UART_Receive(&cmd, 1) == HAL_OK) {
            switch(cmd) {
                case CMD_ERASE:
                    EraseFlash();
                    break;
                    
                case CMD_WRITE:
                    WriteFlash();
                    break;
                    
                case CMD_VERIFY:
                    VerifyFlash();
                    break;
                    
                case CMD_RESET:
                    NVIC_SystemReset();
                    break;
            }
        }
    }
}
```

### Firmware Update Process
```c
void UpdateFirmware(void) {
    // 1. Xóa sector
    FLASH_EraseSector();
    
    // 2. Nhận firmware
    while(receivedSize < totalSize) {
        // Nhận data packet
        ReceivePacket(&packet);
        
        // Ghi vào flash
        WriteFlash(packet);
        
        // Verify
        if(!VerifyPacket(packet)) {
            // Error handling
            RollbackUpdate();
            return;
        }
    }
    
    // 3. Update complete
    UpdateBootFlags();
    SystemReset();
}
```

---

## Các điểm quan trọng

### Best Practices
1. **Luôn có cơ chế rollback**
2. **Verify mọi thao tác write**
3. **Bảo vệ bootloader sector**
4. **Implement watchdog**
5. **Sử dụng CRC check**

### Common Pitfalls
1. Không kiểm tra CRC
2. Thiếu cơ chế recovery
3. Không bảo vệ bootloader
4. Bỏ qua hardware init
5. Thiếu error handling

### Development Tips
1. Sử dụng debug interface
2. Test tất cả scenarios
3. Implement logging
4. Tối ưu boot time
5. Secure communication