# Hướng dẫn Lập trình STM32F411

## Mục lục
1. Giới thiệu về STM32F411
2. Cấu hình và Công cụ phát triển
3. GPIO (General Purpose Input/Output)
4. Timers
5. ADC (Analog-to-Digital Converter)
6. UART/USART Communication
7. I2C Communication
8. SPI Communication
9. DMA (Direct Memory Access)
10. Interrupt và NVIC

---

## 1. Giới thiệu về STM32F411

### Đặc điểm chính:
- Vi xử lý ARM Cortex-M4 với FPU
- Tốc độ xử lý lên đến 100MHz
- Flash memory: 512KB
- SRAM: 128KB
- Nguồn cấp: 1.7V đến 3.6V

### Các tính năng nổi bật:
- 7 Timer 16-bit và 1 Timer 32-bit
- 1 ADC với độ phân giải 12-bit
- Giao tiếp: USART, I2C, SPI
- DMA controller
- Nhiều GPIO pin

---

## 2. Cấu hình và Công cụ phát triển

### Công cụ cần thiết:
1. **STM32CubeIDE**
   - IDE tích hợp cho STM32
   - Bao gồm công cụ cấu hình đồ họa
   - Trình biên dịch GCC

2. **STM32CubeMX**
   - Công cụ cấu hình đồ họa
   - Tạo code khởi tạo
   - Cấu hình clock và pin

### Các bước cài đặt:
1. Tải và cài đặt STM32CubeIDE
2. Tạo project mới
3. Cấu hình cơ bản:
   - System Clock
   - Debug interface
   - GPIO

---

## 3. GPIO (General Purpose Input/Output)

### Cấu hình GPIO:
```c
// Enable GPIO Clock
RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN;

// Configure GPIO Mode
GPIOA->MODER |= GPIO_MODER_MODE5_0;  // Output mode

// Set GPIO Pin
GPIOA->BSRR = GPIO_BSRR_BS5;

// Reset GPIO Pin
GPIOA->BSRR = GPIO_BSRR_BR5;
```

### Các mode hoạt động:
- Input mode
- Output mode
- Alternate function
- Analog mode

---

## 4. Timers

### Các loại Timer:
1. **Basic Timer**
2. **General Purpose Timer**
3. **Advanced Timer**

### Ví dụ cấu hình Timer:
```c
// Enable Timer Clock
RCC->APB1ENR |= RCC_APB1ENR_TIM2EN;

// Set prescaler và period
TIM2->PSC = 8399;  // 100MHz / 8400 = 10KHz
TIM2->ARR = 9999;  // 10KHz / 10000 = 1Hz

// Enable Timer
TIM2->CR1 |= TIM_CR1_CEN;
```

---

## 5. ADC (Analog-to-Digital Converter)

### Đặc điểm ADC:
- Độ phân giải 12-bit
- Tốc độ lấy mẫu 2.4 MSPS
- 16 kênh

### Cấu hình ADC:
```c
// Enable ADC Clock
RCC->APB2ENR |= RCC_APB2ENR_ADC1EN;

// Configure ADC
ADC1->CR2 |= ADC_CR2_ADON;  // Enable ADC
ADC1->SQR3 = 0;  // Channel 0

// Start conversion
ADC1->CR2 |= ADC_CR2_SWSTART;

// Wait for conversion
while(!(ADC1->SR & ADC_SR_EOC));

// Read result
uint16_t result = ADC1->DR;
```

---

## 6. UART/USART Communication

### Cấu hình UART:
```c
// Enable USART Clock
RCC->APB1ENR |= RCC_APB1ENR_USART2EN;

// Configure baudrate
USART2->BRR = 0x0683;  // 9600 baud @ 16MHz

// Enable UART
USART2->CR1 |= USART_CR1_TE | USART_CR1_RE | USART_CR1_UE;
```

### Truyền và nhận dữ liệu:
```c
void UART_SendChar(char c) {
    while(!(USART2->SR & USART_SR_TXE));
    USART2->DR = c;
}

char UART_GetChar(void) {
    while(!(USART2->SR & USART_SR_RXNE));
    return USART2->DR;
}
```

---

## 7. I2C Communication

### Cấu hình I2C:
```c
// Enable I2C Clock
RCC->APB1ENR |= RCC_APB1ENR_I2C1EN;

// Configure I2C
I2C1->CR2 = 16;  // 16MHz peripheral clock
I2C1->CCR = 80;  // 100KHz
I2C1->TRISE = 17;

// Enable I2C
I2C1->CR1 |= I2C_CR1_PE;
```

### Truyền dữ liệu I2C:
```c
void I2C_Write(uint8_t address, uint8_t data) {
    // Wait until I2C is ready
    while(I2C1->SR2 & I2C_SR2_BUSY);
    
    // Send START condition
    I2C1->CR1 |= I2C_CR1_START;
    
    // Wait for START to be sent
    while(!(I2C1->SR1 & I2C_SR1_SB));
    
    // Send slave address
    I2C1->DR = address << 1;
    
    // Wait for address to be sent
    while(!(I2C1->SR1 & I2C_SR1_ADDR));
    
    // Clear ADDR flag
    (void)I2C1->SR2;
    
    // Send data
    I2C1->DR = data;
    
    // Wait for data to be sent
    while(!(I2C1->SR1 & I2C_SR1_BTF));
    
    // Send STOP condition
    I2C1->CR1 |= I2C_CR1_STOP;
}
```

---

## 8. SPI Communication

### Cấu hình SPI:
```c
// Enable SPI Clock
RCC->APB2ENR |= RCC_APB2ENR_SPI1EN;

// Configure SPI
SPI1->CR1 = SPI_CR1_SSM |    // Software slave management
             SPI_CR1_SSI |    // Internal slave select
             SPI_CR1_BR_0 |   // Baud rate control
             SPI_CR1_MSTR;    // Master mode

// Enable SPI
SPI1->CR1 |= SPI_CR1_SPE;
```

### Truyền/nhận dữ liệu SPI:
```c
uint8_t SPI_Transfer(uint8_t data) {
    // Wait until TX buffer is empty
    while(!(SPI1->SR & SPI_SR_TXE));
    
    // Send data
    SPI1->DR = data;
    
    // Wait until RX buffer is not empty
    while(!(SPI1->SR & SPI_SR_RXNE));
    
    // Return received data
    return SPI1->DR;
}
```

---

## 9. DMA (Direct Memory Access)

### Cấu hình DMA:
```c
// Enable DMA Clock
RCC->AHB1ENR |= RCC_AHB1ENR_DMA1EN;

// Configure DMA
DMA1_Stream0->CR = DMA_SxCR_CHSEL_0 |  // Channel selection
                   DMA_SxCR_MINC |      // Memory increment mode
                   DMA_SxCR_DIR_0;      // Direction memory to peripheral

// Set addresses
DMA1_Stream0->PAR = (uint32_t)&peripheral_address;
DMA1_Stream0->M0AR = (uint32_t)&memory_address;

// Set data count
DMA1_Stream0->NDTR = data_size;

// Enable DMA
DMA1_Stream0->CR |= DMA_SxCR_EN;
```

---

## 10. Interrupt và NVIC

### Cấu hình Interrupt:
```c
// Enable EXTI interrupt
NVIC_EnableIRQ(EXTI0_IRQn);

// Configure EXTI
SYSCFG->EXTICR[0] = SYSCFG_EXTICR1_EXTI0_PA;  // PA0
EXTI->IMR |= EXTI_IMR_MR0;  // Unmask
EXTI->RTSR |= EXTI_RTSR_TR0;  // Rising trigger
```

### Xử lý Interrupt:
```c
void EXTI0_IRQHandler(void) {
    if(EXTI->PR & EXTI_PR_PR0) {
        // Clear pending bit
        EXTI->PR = EXTI_PR_PR0;
        
        // Interrupt handling code
    }
}
```

---

## Tài liệu tham khảo

1. [STM32F411 Reference Manual](https://www.st.com/resource/en/reference_manual/dm00119316-stm32f411xc-e-advanced-arm-based-32-bit-mcus-stmicroelectronics.pdf)
2. [STM32F411 Datasheet](https://www.st.com/resource/en/datasheet/stm32f411ce.pdf)
3. [STM32CubeIDE User Manual](https://www.st.com/resource/en/user_manual/dm00603964-stm32cubeide-user-guide-stmicroelectronics.pdf)