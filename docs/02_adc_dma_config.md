
# STM32U5 ADC 采集配置笔记

## 使用的外设
- ADC1, 通道3 (PA0)
- 定时器2触发，采样率 500Hz
- DMA1 循环模式

## 关键代码
```c
// ADC 初始化部分（粘贴关键几行）
HAL_ADC_Start_DMA(&hadc1, (uint32_t*)adc_buffer, BUFFER_SIZE);

知识点结论

Pan-Tompkins 自适应阈值原理：QRS 检测不应靠固定的过零穿越判断，而应维护两个运行估计 —— SPKI（信号峰）和 NPKI（噪声峰），每次检测到 QRS 用 SPKI = 0.125×peak + 0.875×SPKI 更新信号估计，没检测到时用同样公式更新噪声估计。判决阈值 THR = NPKI + 0.25×(SPKI − NPKI) 能自动适应信号幅度变化，避免 T 波误检和弱 QRS 漏检。
LMS 自适应滤波器的关键细节：参考信号 sin(2π×50×n/Fs) 中的 n（离散时间索引）必须每采样点递增，否则 sin/cos 永远输出同一个值，滤波器退化为固定增益放大器，无法跟踪 50Hz 工频。
STM32 FMC LCD 初始化顺序：ctp_test() 触摸屏测试如果不加保护直接调用 tp_dev.scan(0)，而该函数指针为 NULL（未完成 I2C 初始化），会触发 HardFault，导致后续所有代码（包括 LCD 清屏、ECG 采集初始化）全部跳过 —— 表面现象是屏幕灰色。
Python 字节级文件修复：当 IDE 编辑工具遇到中文编码不匹配时，用 Python 直接以二进制模式读写文件、.encode('utf-8') 做字节级查找替换，是处理嵌入式 C 代码中混有中文注释时的可靠方案。
踩坑记录

问题	原因	解决
main.c 编译报 19 个 redefinition 错误	前一轮对话中编辑操作产生了重复的 USER CODE 块（行 364-735 与后面完全重复）	用 Python 脚本按行切片重建：保留 1-363 + LCD 函数段 + 736 以后，验证 46 对 USER CODE 标记全部平衡
LCD 上电后灰屏，无任何显示	ctp_test() 中的 while(1) 死循环 + tp_dev.scan=NULL 触发 HardFault，ECG 和 LCD 初始化全部被跳过	注释掉 main.c 中的 ctp_test() 调用
心率数字 130-150 BPM，明显偏高	T 波被误判为 QRS（过零检测对 T 波敏感）；不应期 FS/3=83 采样点（332ms）太短，挡不住 T 波	① QRS 输入信号从 ×100 改为 ×500 放大 ② 不应期从 FS/3 延长到 FS/2（500ms）
心率在 37-100 之间剧烈跳动	根本原因：过零检测用 temp[wp] > peak_mid 判断，peak_mid 是整个 180 点缓冲区的全局中值，对小幅度 QRS 完全不敏感；阈值变量 threshold_i1 声明了但从未更新，始终为 0	用 Python 脚本 fix_pt_qrs.py 整体替换 QRS 检测段，改为标准自适应阈值（SPKI/NPKI 运行估计 + 动态门限），并设最低阈值 = 10 防止噪声误触发
LMS 滤波器对 50Hz 几乎无抑制效果	filter_index 在 lms_filter_init() 中初始化为 0，但调用 lms_filter() 时每次都是 filter_index=1 → sin/cos 参考信号恒定不变	在 lms_filter() 末尾加 filter_index += 1.0f，确保参考信号按采样率步进
Edit 工具反复报 "old_string not found"	文件中包含中文注释，IDE 的 Edit 工具字符串匹配对 UTF-8 编码处理不稳定	统一改用 Python 脚本做 content.find(old.encode('utf-8')) 字节级替换，不再使用 Edit 工具改中文文件
下一步行动

编译 + 烧录测试：在 Keil MDK 中重新编译项目，烧录到 STM32U5 开发板，观察串口输出的心率是否稳定在 60-90 BPM（静息正常范围）
如果心率仍偏高/偏低：调整缩放系数（当前 ×500）或自适应阈值的最低门限（当前 10），这两个参数直接影响检测灵敏度
后续优化方向：如果自适应阈值工作正常，可以考虑加入 RR 间期平均和异常间期剔除（代码中 rr1[]/rr2[] 数组已声明但未使用），进一步提升心率计算的抗干扰能力
