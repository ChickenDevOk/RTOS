## 5. Timer và Interrupt

### Software Timer
- Không chiếm CPU time
- Callback function chạy trong ngữ cảnh timer task
- One-shot hoặc auto-reload timers

### Timer Creation
```c
TimerHandle_t xTimerCreate(
  const char * const pcTimerName,
  const TickType_t xTimerPeriod,
  const UBaseType_t uxAutoReload,
  void * const pvTimerID,
  TimerCallbackFunction_t pxCallbackFunction
);
```

### Interrupt Handling
- ISR phải ngắn gọn
- Sử dụng "FromISR" API
- Defer processing to tasks

### Ví dụ Timer
```c
void vTimerCallback(TimerHandle_t xTimer) {
  // Timer code here
}

// Create and start timer
TimerHandle_t xTimer = xTimerCreate(
  "MyTimer",
  pdMS_TO_TICKS(1000),
  pdTRUE,  // Auto reload
  0,
  vTimerCallback
);
xTimerStart(xTimer, 0);
```

### Interrupt Safe API
```c
// Trong ISR
void vISR(void) {
  BaseType_t xHigherPriorityTaskWoken = pdFALSE;
  
  // Gửi dữ liệu từ ISR
  xQueueSendFromISR(xQueue,
                    &value,
                    &xHigherPriorityTaskWoken);
                    
  portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}