# FreeRTOS trên STM32F411

## Mục lục
1. Giới thiệu FreeRTOS
2. Kiến trúc FreeRTOS
3. Task Management
4. Queue & Semaphore
5. Interrupt Handling
6. Memory Management
7. Cấu hình FreeRTOS trên STM32F411
8. Ví dụ thực tế

---

## 1. Giới thiệu FreeRTOS

### FreeRTOS là gì?
- Hệ điều hành thời gian thực (RTOS) mã nguồn mở
- Kernel nhỏ gọn, dễ sử dụng
- Hỗ trợ đa nền tảng vi điều khiển
- Được sử dụng rộng rãi trong các ứng dụng nhúng

### Đặc điểm chính
- Lập lịch ưu tiên (Priority-based scheduling)
- Quản lý tài nguyên
- Đồng bộ hóa và giao tiếp giữa các task
- Quản lý thời gian
- Xử lý ngắt hiệu quả

---

## 2. Kiến trúc FreeRTOS

### Kernel Architecture
```
         Application Tasks
  ┌─────────┬─────────┬─────────┐
  │ Task 1  │ Task 2  │ Task N  │
  └────┬────┴────┬────┴────┬────┘
       │         │         │
  ┌────▼─────────▼─────────▼────┐
  │        FreeRTOS Kernel      │
  └────┬─────────┬─────────┬────┘
       │         │         │
  ┌────▼────┐┌───▼────┐┌──▼─────┐
  │ Task    ││ Queue  ││ Timer  │
  │Manager  ││Manager ││Services│
  └─────────┘└────────┘└────────┘
```

### Scheduler Types
1. **Preemptive Scheduling**
  - Task ưu tiên cao hơn có thể chiếm CPU
  - Đảm bảo thời gian đáp ứng
  
2. **Cooperative Scheduling**
  - Task chạy đến khi tự nguyện nhường CPU
  - Ít overhead hơn

---

## 3. Task Management

### Task States
```
                  RUNNING
                 ▲      │
            Select│      │Preempt
                 │      ▼
                 READY◄────┐
                 ▲         │
         Unblock │         │ Block
                 │         │
                 BLOCKED   │
                 ▲         │
      Create    │         │
               │         ▼
            SUSPENDED
```

### Task Creation
```c
BaseType_t xTaskCreate(
    TaskFunction_t pvTaskCode,
    const char * pcName,
    uint16_t usStackDepth,
    void *pvParameters,
    UBaseType_t uxPriority,
    TaskHandle_t *pxCreatedTask
);
```

### Ví dụ Task
```c
void vTaskFunction(void *pvParameters) {
    for(;;) {
        // Task code here
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

---

## 4. Queue & Semaphore

### Queue
```c
// Tạo Queue
QueueHandle_t xQueue = xQueueCreate(5, sizeof(uint32_t));

// Gửi dữ liệu
uint32_t value = 100;
xQueueSend(xQueue, &value, portMAX_DELAY);

// Nhận dữ liệu
uint32_t receivedValue;
xQueueReceive(xQueue, &receivedValue, portMAX_DELAY);
```

### Semaphore
```c
// Binary Semaphore
SemaphoreHandle_t xSemaphore = xSemaphoreCreateBinary();

// Take semaphore
if(xSemaphoreTake(xSemaphore, portMAX_DELAY) == pdTRUE) {
    // Critical section
    xSemaphoreGive(xSemaphore);
}
```

### Mutex
```c
// Create mutex
SemaphoreHandle_t xMutex = xSemaphoreCreateMutex();

// Use mutex
if(xSemaphoreTake(xMutex, portMAX_DELAY) == pdTRUE) {
    // Protected resource access
    xSemaphoreGive(xMutex);
}
```

---

## 5. Interrupt Handling

### ISR Safe Functions
```c
// Từ ISR
BaseType_t xHigherPriorityTaskWoken = pdFALSE;

xQueueSendFromISR(xQueue, &value, &xHigherPriorityTaskWoken);

portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
```

### Deferred Interrupt Processing
```c
void vDeferredHandlingTask(void *pvParameters) {
    for(;;) {
        // Đợi notification từ ISR
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);
        // Xử lý interrupt
    }
}
```

---

## 6. Memory Management

### Heap Models
1. **heap_1**
  - Phân bổ đơn giản nhất
  - Không hỗ trợ free memory
  
2. **heap_2**
  - Best fit algorithm
  - Có thể gây phân mảnh
  
3. **heap_3**
  - Sử dụng malloc/free của C
  - Thread safe
  
4. **heap_4**
  - First fit với merge
  - Hiệu quả nhất
  
5. **heap_5**
  - Giống heap_4
  - Hỗ trợ nhiều vùng nhớ

### Memory Allocation
```c
void *pvPortMalloc(size_t xSize);
void vPortFree(void *pv);
```

---

## 7. Cấu hình FreeRTOS trên STM32F411

### FreeRTOSConfig.h
```c
// Cấu hình cơ bản
#define configUSE_PREEMPTION                    1
#define configUSE_PORT_OPTIMISED_TASK_SELECTION 0
#define configUSE_TICKLESS_IDLE                 0
#define configCPU_CLOCK_HZ                      100000000
#define configTICK_RATE_HZ                      1000
#define configMAX_PRIORITIES                    5
#define configMINIMAL_STACK_SIZE               128
#define configTOTAL_HEAP_SIZE                  8192
#define configMAX_TASK_NAME_LEN                16
```

### System Timer Configuration
```c
void vPortSetupTimerInterrupt(void) {
    // Cấu hình SysTick cho RTOS tick
    SysTick_Config(SystemCoreClock / configTICK_RATE_HZ);
}
```

---

## 8. Ví dụ thực tế

### Blink LED với FreeRTOS
```c
void LED_Task(void *pvParameters) {
    // Cấu hình GPIO
    RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN;
    GPIOA->MODER |= GPIO_MODER_MODE5_0;
    
    for(;;) {
        GPIOA->ODR ^= GPIO_ODR_OD5;
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

int main(void) {
    // Khởi tạo system clock
    SystemClock_Config();
    
    // Tạo task
    xTaskCreate(LED_Task, "LED", 128, NULL, 1, NULL);
    
    // Khởi động scheduler
    vTaskStartScheduler();
    
    while(1);
}
```

### UART Communication Task
```c
void UART_Task(void *pvParameters) {
    char rxData;
    
    // Khởi tạo UART
    UART_Init();
    
    for(;;) {
        if(xQueueReceive(uartRxQueue, &rxData, portMAX_DELAY)) {
            // Xử lý dữ liệu nhận được
            Process_UART_Data(rxData);
        }
    }
}
```

### Sensor Reading Task
```c
void Sensor_Task(void *pvParameters) {
    uint16_t sensorValue;
    
    for(;;) {
        // Đọc giá trị cảm biến
        sensorValue = Read_ADC();
        
        // Gửi giá trị vào queue
        xQueueSend(sensorQueue, &sensorValue, portMAX_DELAY);
        
        // Delay 100ms
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

---

## Tài liệu tham khảo

1. [FreeRTOS Official Documentation](https://www.freertos.org/Documentation/RTOS_book.html)
2. [STM32 FreeRTOS Examples](https://github.com/STMicroelectronics/STM32CubeF4)
3. [FreeRTOS API Reference](https://www.freertos.org/a00106.html)