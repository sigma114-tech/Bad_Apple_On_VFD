# Bad_Apple_On_VFD
把 48×256 的 Bad Apple 逐帧图像，经由 Python 串口 → STM32F103C8T6 → SPI(DMA) 推送到 VFD 控制器 实时播放。
目标帧率 30 FPS，整链路采用 UART DMA 双缓冲接收 + SPI DMA 发送，CPU 占用极低。

功能特性

图片离线预处理：将 48×256 灰度帧转为 1536B/帧（每字节 8 像素，列优先）。

MCU 侧：

USART1 + DMA：每帧 1536 字节双缓冲接收；

SPI1 + DMA：一帧数据一次 DMA 推送到 VFD；

初始化/清屏/亮度设置/显示模式/偏移等基础指令封装。

可选握手：SPI DMA 完成后 MCU 回 0xA5，PC 收到再发下一帧，避免卡顿与抖动。

硬件与环境

MCU：STM32F103C8T6（Blue Pill 或等效板）

显示：VFD 控制器（SPI 接口，命令兼容本文档示例）

连接（示例）

功能	STM32 引脚	备注
SPI1_SCK	PA5	上拉/终端按需
SPI1_MOSI	PA7	1-Line TxOnly 推荐
VFD_CS/LATCH	你配置的 GPIO	片选/锁存
VFD_RESET	你配置的 GPIO	低复位
USART1_TX/RX	PA9/PA10	与 PC 串口工具连接
供电	按模块规格	VFD 灯丝/高压参考其手册

开发环境

ARM-GCC for STM32（CubeIDE 或 gnu-tools-for-stm32）

CMake + Ninja（VS Code/CMake Tools 预设）

Python 3.10+（opencv-python, numpy, pyserial）

目录结构
.
├─ Core/                 # CubeMX 生成代码（main、外设 init、ISR 等）
├─ Drivers/              # HAL/CMSIS
├─ cmake/                # CMake 构建脚本（含 stm32cubemx）
├─ scripts/              # Python 工具：编码/发送
│  ├─ encode_frames.py   # 将 48×256 图像转 1536B/帧（二值化→逐字节）
│  └─ send_serial.py     # 串口发送帧（支持握手）
├─ Image/                # 输入图片（badapple0000.jpg ~ ...）
├─ Image_Encode/         # 输出帧二进制（Image0000.bin ...）
├─ CMakeLists.txt
├─ CMakePresets.json
├─ PROJECT.ioc           # STM32CubeMX 配置
└─ README.md

快速开始
1) 准备帧数据（Python）
pip install opencv-python numpy pyserial
python scripts/encode_frames.py  # 读取 Image/*.jpg，生成 Image_Encode/*.bin（每帧1536B）


二值阈值默认为 128；

打包规则：列优先（x:0→255），每列 6 字节；每字节 bit0 为块内最上像素，bit7 为最下像素。

2) 构建固件（CMake）
cmake --preset Release
cmake --build --preset Release


生成的固件在 build/Release/（.elf/.bin/.hex）。

3) 烧录

使用 ST-Link 或 STM32CubeProgrammer 烧录到目标板。

4) 运行（发送Bad Apple）

推荐握手模式（最稳）：MCU 在 SPI DMA 完成后通过 UART 回 0xA5，PC 脚本等到 0xA5 再发下一帧，无需 sleep。

串口波特率建议 ≥ 921600；若仅 512000，可在脚本里设置 sleep≈5–8ms 保持稳定（见脚本注释）。

python scripts/send_serial.py --port COM11 --baud 921600 --dir Image_Encode

协议/数据格式

分辨率：48×256（高×宽）

帧格式：1536 字节/帧

每列 48 像素 → 分成 6 个 8 像素块；每块打包成 1 字节（bit0 顶、bit7 底）

列从左到右；每列内从上到下打包

MCU SPI 发送顺序：逐字节线性发送 1536 字节

可选 ACK：每帧 SPI DMA 完成后 MCU 发送 0xA5

性能建议

SPI：1-Line(Tx Only)、NSS=Software（SSM=1/SSI=1），分频 4–16 先跑通再提速

UART：≥921600bps（30 FPS 余量足）；512000bps 时建议发送端 sleep≈5ms 或使用握手

DMA：USART1_RX（双缓冲思路）、SPI1_TX；回调里仅做轻量操作（拉 CS、清忙、回 ACK）

常见问题

画面慢/抖：提升波特率；去掉固定 sleep，启用 ACK；或小幅 sleep(5–8ms) 于 512000bps。

SPI TXE 卡住：确认 SPE=1、SSM=1/SSI=1，主模式 MODF 未触发；推荐 1-Line(Tx Only)。

黑白反/上下翻：在编码脚本中反转位序或在固件中取反。

许可证

本项目采用 MIT License（见 LICENSE 文件）。
你可以自由地使用、修改与分发本项目代码与二进制。

致谢

ST 官方 HAL/CMSIS 与工具链

开源社区的 CMake、VS Code 插件以及示例

上位机帧传输思路参考自 @WanDejun 的 STM32F103C8T6 仓库（BadApple 目录相关讨论与实践）。
https://github.com/WanDejun/STM32F103C8T6

显示硬件资料参考 OSHWHub – GP1287BI VFD256×50 模块（UNL-200AP）：
https://oshwhub.com/XACT/gp1287bi-vfd-xian-shi-mu-kuai

经典作品 Bad Apple!! 带来的灵感
