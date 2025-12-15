# rnn-denoise 单片机部署保姆级流程

面向第一次接触 NNoM 的用户，按顺序完成下列步骤即可将 `examples/rnn-denoise` 实时降噪示例跑在 Cortex-M 系列 MCU（如 STM32L476）上。

## 0. 准备环境
- 安装 Python 3.8+，确保 `pip` 可用。
- 推荐在 PC 上先完成训练和量化；MCU 端只编译推理。 
- 克隆本仓库后进入 `examples/rnn-denoise/` 目录，后续命令默认在该目录执行。

## 1. 安装依赖并生成训练数据
1. 安装 Python 依赖：
   ```bash
   pip install -r ../../requirements.dev.txt
   ```
2. 生成带噪声的语音数据与均衡器系数（会得到 `equalizer_coeff.h`）：
   ```bash
   python gen_dataset.py
   ```

## 2. 训练并导出 MCU 权重
运行训练与量化脚本，得到 MCU 侧需要的模型权重头文件：
```bash
python main.py
```
输出的 `denoise_weights.h`（若开启均衡器则还有 `equalizer_coeff.h`）位于当前目录。它们会被 `main_arm.c` 在 MCU 上直接包含，文件名与位置不需要改动即可编译通过。

## 3. MCU 工程准备
1. 复制以下文件到你的 MCU 工程同一目录或配置好包含路径：
   - `main_arm.c`：MCU 示例主程序（音频 DMA→MFCC→RNN 推理→均衡器）。
   - `mfcc.c` / `mfcc.h`：MFCC 特征计算实现。
   - `wav.h`：音频缓冲和格式定义。
   - 训练阶段生成的 `denoise_weights.h`、`equalizer_coeff.h`。
2. 保持 `mfcc.h` 中的宏定义与训练时一致，例如滤波器数量 `NUM_FILTER`、FFT 点数等；如果修改这些配置，需要重新运行训练脚本生成匹配的权重与系数。

## 4. 打开平台优化
- 目标为 ARM Cortex-M 时，请在 `mfcc.h` 中启用 ARM 专用优化（定义 `PLATFORM_ARM`），并在工程中链接 CMSIS-NN、ARM FFT 库以提升速度。
- 确保编译器打开 FPU 与 `-O2`/`-O3` 优化选项。

## 5. 连接音频外设
`main_arm.c` 已包含使用板载麦克风和 LED 指示的示例流程。如果目标板的音频采集/播放接口不同：
- 替换或适配 DMA/I2S/SAI 初始化与中断回调，使其填充示例中的输入缓冲。
- 如果需要将降噪后的音频播放或送出，请在 `main_arm.c` 的输出缓冲处理处接入你的 DAC / I2S 发送逻辑。

## 6. 编译与烧录
1. 将上述源文件加入工程，确保包含路径能找到生成的头文件。
2. 使用厂商工具链（如 STM32CubeIDE、Keil、IAR）编译并烧录到开发板。
3. 上电后，程序会持续：采集音频 → 计算 MFCC → 在 MCU 上运行 RNN 推理 → 调整均衡器系数 → 输出降噪音频。示例中会通过 LED 指示 VAD 结果。

## 7. 性能与资源提示
- STM32L476（140 MHz M4F）在官方配置下可在 16 ms 帧内完成全部处理，适合实时应用；更低规格 MCU 可调小滤波器数量或降低采样率以换取速度。
- 通过减小模型规模或关闭均衡器可进一步节省 RAM/Flash 占用。

## 8. 常见问题
- **训练生成的头文件放哪里？** 与 `main_arm.c` 同级即可，或在工程中加入相应头文件搜索路径。
- **修改滤波器数量/采样率后出错？** 需同步更新 `mfcc.h` 宏并重新运行 `gen_dataset.py`、`main.py` 生成新的系数和权重。
- **如何验证效果？** 训练脚本会在 PC 端生成 `_nn_fixedpoit_filtered_sample.wav` 等文件，可先在 PC 上听效果，再烧录到板上测试实采音频。
