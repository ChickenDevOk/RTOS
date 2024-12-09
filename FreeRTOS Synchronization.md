## 4. Semaphore và Mutex

### Binary Semaphore
- Giống như mutex nhưng dùng cho đồng bộ hóa
- Thường dùng với interrupt
- Chỉ có giá trị 0 hoặc 1

### Counting Semaphore
- Có thể đếm nhiều hơn 1
- Dùng để quản lý tài nguyên
- Theo dõi sự kiện

### Mutex
- Dùng cho mutual exclusion
- Priority inheritance
- Tránh priority inversion

### Ví dụ Mutex
```c
// Tạo mutex
SemaphoreHandle_t xMutex;
xMutex = xSemaphoreCreateMutex();

// Sử dụng mutex
void vTask(void *pvParameters) {
  while(1) {
      if(xSemaphoreTake(xMutex, portMAX_DELAY)) {
          // Critical section
          xSemaphoreGive(xMutex);
      }
      vTaskDelay(pdMS_TO_TICKS(100));
  }
}
```

### Ví dụ Binary Semaphore với Interrupt
```c
void vHandlerTask(void *pvParameters) {
  while(1) {
      xSemaphoreTake(xBinarySemaphore, portMAX_DELAY);
      // Handle interrupt event
  }
}

void vISR(void) {
  BaseType_t xHigherPriorityTaskWoken = pdFALSE;
  xSemaphoreGiveFromISR(xBinarySemaphore, 
                       &xHigherPriorityTaskWoken);
  portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}