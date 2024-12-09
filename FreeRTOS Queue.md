## 3. Queue và Communication

### Queue
- Cơ chế giao tiếp giữa các task
- FIFO (First In First Out)
- Thread-safe
- Có thể chứa nhiều kiểu dữ liệu

### Queue Operations
```c
// Tạo queue
QueueHandle_t xQueueCreate(
  UBaseType_t uxQueueLength,
  UBaseType_t uxItemSize
);

// Gửi dữ liệu
xQueueSend(QueueHandle_t xQueue,
         const void *pvItemToQueue,
         TickType_t xTicksToWait);

// Nhận dữ liệu
xQueueReceive(QueueHandle_t xQueue,
            void *pvBuffer,
            TickType_t xTicksToWait);
```

### Ví dụ sử dụng Queue
```c
// Sender Task
void vSenderTask(void *pvParameters) {
  int32_t value = 0;
  while(1) {
      xQueueSend(xQueue, &value, portMAX_DELAY);
      value++;
      vTaskDelay(pdMS_TO_TICKS(1000));
  }
}

// Receiver Task
void vReceiverTask(void *pvParameters) {
  int32_t receivedValue;
  while(1) {
      if(xQueueReceive(xQueue, &receivedValue, portMAX_DELAY)) {
          printf("Received: %d\n", receivedValue);
      }
  }
}
```