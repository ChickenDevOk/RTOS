## 6. Memory Management

### Heap Management
- 5 scheme khác nhau (heap_1 đến heap_5)
- Các cách cấp phát khác nhau
- Trade-off giữa tính năng và độ phức tạp

### Heap Schemes
1. **heap_1**: Đơn giản nhất, không free memory
2. **heap_2**: Cho phép free, best fit
3. **heap_3**: Wrapper cho malloc/free
4. **heap_4**: Kết hợp các block liền kề
5. **heap_5**: Giống heap_4 nhưng nhiều vùng nhớ

### Memory Allocation
```c
// Cấp phát
void *pvPortMalloc(size_t xSize);

// Giải phóng
void vPortFree(void *pv);

// Kiểm tra bộ nhớ còn lại
size_t xPortGetFreeHeapSize(void);
```

### Best Practices
- Tránh cấp phát động trong runtime
- Sử dụng static allocation khi có thể
- Kiểm tra memory leaks
- Theo dõi heap usage