# Bootloader - Giải thích chi tiết

## 1. Cấu trúc bộ nhớ và Vector Table

### Memory Map chi tiết
```
Flash Memory
┌─────────────────────────┐ 0x0800 0000
│    Vector Table         │
│    (Bootloader)         │
├─────────────────────────┤ 
│    Bootloader Code      │
│                         │
├─────────────────────────┤ 0x0800 8000
│    Vector Table         │
│    (Application)        │
├─────────────────────────┤
│    Application Code     │
│                         │
├─────────────────────────┤ 0x0808 0000
│    Configuration        │
│    Parameters           │
└─────────────────────────┘ 0x0808 8000
```

### Vector Table Structure
```c
__attribute__((section(".isr_vector")))
const uint32_t g_pfnVectors[] = {
    // Stack pointer value
    (uint32_t)&_estack,
    
    // Core vectors
    (uint32_t)Reset_Handler,
    (uint32_t)NMI_Handler,
    (uint32_t)HardFault_Handler,
    (uint32_t)MemManage_Handler,
    (uint32_t)BusFault_Handler,
    // ...
    
    // Peripheral vectors
    (uint32_t)WWDG_IRQHandler,
    (uint32_t)PVD_IRQHandler,
    // ...
};
```

---

## 2. Quá trình khởi động chi tiết

### Reset Sequence
```
Power On/Reset
     │
     ▼
1. Load Stack Pointer
     │
     ▼
2. Jump to Reset Handler
     │
     ▼
3. System Initialization
     │
     ▼
4. Clock Configuration
     │
     ▼
5. Peripherals Init
     │
     ▼
6. Boot Decision
```

### Clock Configuration
```c
void SystemClock_Config(void) {
    RCC_OscInitTypeDef RCC_OscInitStruct = {0};
    RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};
    
    // Configure HSE
    RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
    RCC_OscInitStruct.HSEState = RCC_HSE_ON;
    RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
    RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;
    RCC_OscInitStruct.PLL.PLLM = 8;
    RCC_OscInitStruct.PLL.PLLN = 336;
    RCC_OscInitStruct.PLL.PLLP = RCC_PLLP_DIV2;
    RCC_OscInitStruct.PLL.PLLQ = 7;
    HAL_RCC_OscConfig(&RCC_OscInitStruct);
    
    // Configure Clock
    RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_SYSCLK | 
                                 RCC_CLOCKTYPE_HCLK | 
                                 RCC_CLOCKTYPE_PCLK1 | 
                                 RCC_CLOCKTYPE_PCLK2;
    RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
    RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
    RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV4;
    RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV2;
    HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_5);
}
```

---

## 3. Cơ chế bảo vệ và xác thực

### CRC Calculation
```c
uint32_t CalculateCRC(uint8_t *data, uint32_t length) {
    uint32_t crc = 0xFFFFFFFF;
    
    // Reset CRC peripheral
    __HAL_RCC_CRC_CLK_ENABLE();
    CRC->CR = CRC_CR_RESET;
    
    // Calculate CRC
    for(uint32_t i = 0; i < length/4; i++) {
        CRC->DR = ((uint32_t*)data)[i];
    }
    
    return CRC->DR;
}
```

### Firmware Verification
```c
typedef struct {
    uint32_t magic;          // Magic number
    uint32_t version;        // Firmware version
    uint32_t size;          // Firmware size
    uint32_t crc;           // CRC value
    uint8_t signature[256]; // RSA signature
} FirmwareHeader_t;

bool VerifyFirmware(uint32_t addr) {
    FirmwareHeader_t *header = (FirmwareHeader_t*)addr;
    
    // Check magic number
    if(header->magic != FIRMWARE_MAGIC)
        return false;
    
    // Verify size
    if(header->size > MAX_FIRMWARE_SIZE)
        return false;
    
    // Calculate CRC
    uint32_t calc_crc = CalculateCRC(
        (uint8_t*)(addr + sizeof(FirmwareHeader_t)),
        header->size
    );
    
    if(calc_crc != header->crc)
        return false;
    
    // Verify signature
    return VerifySignature(header);
}
```

---

## 4. Giao thức truyền firmware

### Packet Structure
```
┌────┬────┬────┬──────┬────┬─────┐
│SOF │LEN │CMD │DATA  │CRC │EOF  │
│1B  │2B  │1B  │0-1KB │2B  │1B   │
└────┴────┴────┴──────┴────┴─────┘
```

### Protocol Implementation
```c
typedef struct {
    uint8_t sof;      // Start of frame = 0x5A
    uint16_t length;  // Packet length
    uint8_t command;  // Command ID
    uint8_t data[1024]; // Data buffer
    uint16_t crc;     // CRC16
    uint8_t eof;      // End of frame = 0xA5
} Packet_t;

void ProcessPacket(void) {
    Packet_t packet;
    HAL_StatusTypeDef status;
    
    // Receive SOF
    status = UART_Receive(&packet.sof, 1, 100);
    if(status != HAL_OK || packet.sof != 0x5A)
        return;
    
    // Receive length
    UART_Receive((uint8_t*)&packet.length, 2, 100);
    
    // Receive command
    UART_Receive(&packet.command, 1, 100);
    
    // Receive data
    if(packet.length > 0)
        UART_Receive(packet.data, packet.length, 1000);
    
    // Receive CRC
    UART_Receive((uint8_t*)&packet.crc, 2, 100);
    
    // Verify CRC
    uint16_t calc_crc = CalculateCRC16(
        (uint8_t*)&packet.command,
        packet.length + 1
    );
    
    if(calc_crc != packet.crc) {
        SendNACK();
        return;
    }
    
    // Process command
    ProcessCommand(&packet);
}
```

---

## 5. Flash Programming

### Sector Operations
```c
HAL_StatusTypeDef EraseSector(uint32_t sector) {
    FLASH_EraseInitTypeDef EraseInitStruct;
    uint32_t SectorError;
    
    HAL_FLASH_Unlock();
    
    EraseInitStruct.TypeErase = FLASH_TYPEERASE_SECTORS;
    EraseInitStruct.VoltageRange = FLASH_VOLTAGE_RANGE_3;
    EraseInitStruct.Sector = sector;
    EraseInitStruct.NbSectors = 1;
    
    if(HAL_FLASHEx_Erase(&EraseInitStruct, &SectorError) != HAL_OK) {
        HAL_FLASH_Lock();
        return HAL_ERROR;
    }
    
    HAL_FLASH_Lock();
    return HAL_OK;
}
```

### Programming Functions
```c
HAL_StatusTypeDef WriteFlash(uint32_t addr, uint8_t *data, uint32_t len) {
    HAL_StatusTypeDef status = HAL_OK;
    
    HAL_FLASH_Unlock();
    
    // Program flash word by word
    for(uint32_t i = 0; i < len; i += 4) {
        status = HAL_FLASH_Program(
            FLASH_TYPEPROGRAM_WORD,
            addr + i,
            *(uint32_t*)(data + i)
        );
        
        if(status != HAL_OK)
            break;
    }
    
    HAL_FLASH_Lock();
    return status;
}
```

---

## 6. Application Management

### Application Header
```c
typedef struct {
    uint32_t stack_addr;     // Stack pointer
    uint32_t reset_handler;  // Reset handler
    uint32_t version;        // App version
    uint32_t size;           // App size
    uint32_t crc;           // App CRC
    uint8_t valid;          // Validity flag
} AppHeader_t;
```

### Application Validation
```c
bool ValidateApplication(uint32_t app_addr) {
    AppHeader_t *header = (AppHeader_t*)app_addr;
    
    // Check stack pointer
    if(header->stack_addr < RAM_START || 
       header->stack_addr > RAM_END)
        return false;
    
    // Check reset handler
    if(header->reset_handler < app_addr || 
       header->reset_handler > FLASH_END)
        return false;
    
    // Verify CRC
    uint32_t calc_crc = CalculateCRC(
        (uint8_t*)(app_addr + sizeof(AppHeader_t)),
        header->size
    );
    
    if(calc_crc != header->crc)
        return false;
    
    return true;
}
```

### Application Launch
```c
void LaunchApplication(uint32_t app_addr) {
    // Disable interrupts
    __disable_irq();
    
    // Reset peripherals
    HAL_DeInit();
    
    // Relocate vector table
    SCB->VTOR = app_addr;
    
    // Get reset handler address
    uint32_t reset_handler = *((uint32_t*)(app_addr + 4));
    
    // Set stack pointer
    __set_MSP(*((uint32_t*)app_addr));
    
    // Jump to application
    ((void(*)())reset_handler)();
}
```

---

## 7. Error Handling và Recovery

### Watchdog Implementation
```c
void ConfigureWatchdog(void) {
    // Initialize IWDG
    IWDG_HandleTypeDef hiwdg;
    hiwdg.Instance = IWDG;
    hiwdg.Init.Prescaler = IWDG_PRESCALER_256;
    hiwdg.Init.Reload = 0xFFF;
    HAL_IWDG_Init(&hiwdg);
}

// Refresh watchdog in main loop
void RefreshWatchdog(void) {
    HAL_IWDG_Refresh(&hiwdg);
}
```

### Error Recovery
```c
typedef enum {
    ERR_NONE = 0,
    ERR_FLASH_ERASE,
    ERR_FLASH_WRITE,
    ERR_CRC,
    ERR_INVALID_APP
} ErrorCode_t;

void HandleError(ErrorCode_t error) {
    // Log error
    LogError(error);
    
    // Increment error counter
    SaveErrorCount();
    
    if(GetErrorCount() > MAX_ERRORS) {
        // Restore backup firmware
        RestoreBackupFirmware();
    }
    
    // Reset system
    NVIC_SystemReset();
}
```

### Backup Management
```c
void BackupFirmware(void) {
    // Copy current firmware to backup sector
    uint32_t *src = (uint32_t*)APP_ADDRESS;
    uint32_t *dst = (uint32_t*)BACKUP_ADDRESS;
    
    EraseSector(BACKUP_SECTOR);
    
    for(uint32_t i = 0; i < APP_SIZE/4; i++) {
        WriteFlash((uint32_t)&dst[i], (uint8_t*)&src[i], 4);
    }
}

void RestoreBackupFirmware(void) {
    // Copy backup firmware to application sector
    uint32_t *src = (uint32_t*)BACKUP_ADDRESS;
    uint32_t *dst = (uint32_t*)APP_ADDRESS;
    
    EraseSector(APP_SECTOR);
    
    for(uint32_t i = 0; i < APP_SIZE/4; i++) {
        WriteFlash((uint32_t)&dst[i], (uint8_t*)&src[i], 4);
    }
}
```

---

## 8. Bootloader Security

### Encryption Implementation
```c
void EncryptFirmware(uint8_t *data, uint32_t size) {
    CRYP_HandleTypeDef hcryp;
    
    // Configure AES
    hcryp.Instance = CRYP;
    hcryp.Init.DataType = CRYP_DATATYPE_8B;
    hcryp.Init.KeySize = CRYP_KEYSIZE_256B;
    hcryp.Init.pKey = aes_key;
    hcryp.Init.pInitVect = aes_iv;
    
    HAL_CRYP_Init(&hcryp);
    
    // Encrypt data
    HAL_CRYP_