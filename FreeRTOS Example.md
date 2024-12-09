## 7. Ví dụ thực tế

### Ví dụ: Hệ thống đo nhiệt độ và hiển thị LCD

```c
// Định nghĩa handle
TaskHandle_t xTempTask;
TaskHandle_t xDisplayTask;
QueueHandle_t xTempQueue;
SemaphoreHandle_t xI2CMutex;

// Task đọc nhiệt độ
void vTempTask(void *pvParameters) {
  float temperature;
  while(1) {
      if(xSemaphoreTake(xI2CMutex, portMAX_DELAY)) {
          temperature = readTemperature();  // Đọc từ sensor
          xSemaphoreGive(xI2CMutex);
          
          xQueueSend(xTempQueue, &temperature, portMAX_DELAY);
          vTaskDelay(pdMS_TO_TICKS(1000));
      }
  }
}

// Task hiển thị
void vDisplayTask(void *pvParameters) {
  float temperature;
  char str[16];
  while(1) {
      if(xQueueReceive(xTempQueue, &temperature, portMAX_DELAY)) {
          if(xSemaphoreTake(xI2CMutex, portMAX_DELAY)) {
              sprintf(str, "Temp: %.1fC", temperature);
              LCD_DisplayString(0, 0, str);
              xSemaphoreGive(xI2CMutex);
          }
      }
  }
}

// Khởi tạo hệ thống
void vSystemInit(void) {
  // Tạo queue và mutex
  xTempQueue = xQueueCreate(1, sizeof(float));
  xI2CMutex = xSemaphoreCreateMutex();
  
  // Tạo tasks
  xTaskCreate(vTempTask, "Temp", configMINIMAL_STACK_SIZE,
              NULL, 2, &xTempTask);
  xTaskCreate(vDisplayTask, "Display", configMINIMAL_STACK_SIZE,
              NULL, 1, &xDisplayTask);
              
  // Bắt đầu scheduler
  vTaskStartScheduler();
}
```

### Các điểm quan trọng trong ví dụ
1. Sử dụng mutex để bảo vệ I2C bus
2. Queue để truyền dữ liệu giữa các task
3. Task priority phù hợp với yêu cầu
4. Delay để giảm tải CPU
5. Error handling (không hiển thị trong ví dụ)