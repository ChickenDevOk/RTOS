## 2. Kiến trúc và thành phần cơ bản

### Kernel Architecture
- Scheduler (Bộ lập lịch)
- Task Management
- Queue Management
- Timer Management
- Interrupt Handling
- Memory Management

### Task States
1. **Running**: Đang thực thi
2. **Ready**: Sẵn sàng thực thi
3. **Blocked**: Đang chờ sự kiện/tài nguyên
4. **Suspended**: Tạm dừng
5. **Deleted**: Đã xóa

### Task Priority
- Mức ưu tiên từ 0 (thấp nhất) đến configMAX_PRIORITIES-1
- Task có ưu tiên cao hơn sẽ được thực thi trước
- Có thể thay đổi ưu tiên trong runtime

### Task Creation
```c
BaseType_t xTaskCreate(
  TaskFunction_t pvTaskCode,
  const char * const pcName,
  uint16_t usStackDepth,
  void *pvParameters,
  UBaseType_t uxPriority,
  TaskHandle_t *pxCreatedTask
);
```

### Ví dụ Task đơn giản
```c
void vTaskFunction(void *pvParameters) {
  while(1) {
      // Task code here
      vTaskDelay(pdMS_TO_TICKS(1000));
  }
}