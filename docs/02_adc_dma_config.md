
# STM32U5 ADC 采集配置笔记

## 使用的外设
- ADC1, 通道3 (PA0)
- 定时器2触发，采样率 500Hz
- DMA1 循环模式

## 关键代码
```c
// ADC 初始化部分（粘贴关键几行）
HAL_ADC_Start_DMA(&hadc1, (uint32_t*)adc_buffer, BUFFER_SIZE);
