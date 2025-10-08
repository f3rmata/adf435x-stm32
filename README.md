# ADF4351 Driver for STM32

> Forked from [original project](https://github.com/kb3gtn/STM32_ADF4351).  
> This repository provides a minimal ADF4350/ADF4351 RF PLL driver based on STM32 HAL (SPI + GPIO).

## 目录
- 简介
- 硬件连接
- 依赖与移植说明
- 快速开始
- API 说明
- 初始化参数详解
- 频率规划与限制
- 常见问题
- 许可证

---

## 简介
本仓库包含对 Analog Devices ADF4350/ADF4351 可编程频率合成器的驱动，适用于 STM32 平台（HAL 库）。核心文件：
- adf4351.h：寄存器位定义、初始化结构体与对外 API 声明
- adf4351.c：基于 SPI 和 GPIO 的底层寄存器写入与频率计算逻辑

驱动支持：
- 分数-N/整数-N 工作模式（根据参数自动配置）
- 参考倍频（RMULT2）、参考二分频（RDIV2）
- 频道间隔（Channel Spacing）设定
- 上电频率与运行时调频
- 主输出功率与辅输出（Aux OUT）控制
- 锁相检测（MUXOUT 可设置为数字锁定指示）

---

## 硬件连接
本驱动默认使用 SPI1 和两个 GPIO 引脚控制 ADF4351 的 LE 与 CE。连接关系如下：
- SPI1_SCK -> ADF4351 CLK
- SPI1_MOSI -> ADF4351 DATA
- ADF4351_LE -> ADF4351 LE
- ADF4351_CE -> ADF4351 CE

注意事项：
- 本驱动不使用 MISO（单向写寄存器）。
- LE/CE 的 GPIO 名称与引脚需在 CubeMX 里定义，生成在 main.h 中，并与 adf4351.c 中使用的宏一致：
  - ADF4351_LE_GPIO_Port / ADF4351_LE_Pin
  - ADF4351_CE_GPIO_Port / ADF4351_CE_Pin
- SPI 模式：CPOL = 0, CPHA = 0（SPI Mode 0），MSB first。常见 SPI 频率 1~10 MHz 均可。

---

## 依赖与移植说明
- 依赖 STM32 HAL：
  - 需提供 SPI_HandleTypeDef hspi1（在用户工程的 spi.c/spi.h 中）
  - 需在 main.h 里定义 ADF4351_LE/CE 的 GPIO 宏
- 如果你使用的不是 SPI1：
  - 修改 adf4351.c 顶部的 extern SPI_HandleTypeDef hspi1 为实际句柄
- 如果你的引脚命名不同：
  - 修改 SPI_assert_CS_LD / SPI_deassert_CS_LD 中使用的 GPIO 宏
- 工程结构：
  - 将 adf4351.c/.h 添加到你的 STM32 工程并参与编译
  - 确保包含路径能找到 adf4351.h 与 main.h、spi.h

---

## 快速开始

示例：上电配置并输出 100 MHz

```c
#include "adf4351.h"

void rf_init(void) {
    adf4350_init_param pll_config;

    // 1) 初始化参数（按需修改）
    pll_config.clkin = 25e6;                 // 外部参考 25 MHz
    pll_config.channel_spacing = 100;        // 频道间隔 100 Hz
    pll_config.power_up_frequency = 50e6;    // 上电频率 50 MHz
    pll_config.reference_div_factor = 1;     // R 分频（1 表示不分频）
    pll_config.reference_doubler_enable = 0; // 参考倍频 x2：0=关 1=开
    pll_config.reference_div2_enable = 0;    // 参考二分频 /2：0=关 1=开

    // R2 用户设置
    pll_config.phase_detector_polarity_positive_enable = 1;
    pll_config.lock_detect_precision_6ns_enable = 1;         // 6ns
    pll_config.lock_detect_function_integer_n_enable = 0;    // 分数-N
    pll_config.charge_pump_current = 2500;                   // uA (312~5000 步进 312)
    pll_config.muxout_select = 6;                            // 6: Digital Lock Detect
    pll_config.low_spur_mode_enable = 0;                     // 低杂散模式

    // R3 用户设置
    pll_config.cycle_slip_reduction_enable = 0;
    pll_config.charge_cancellation_enable = 0;               // ADF4351 专有
    pll_config.anti_backlash_3ns_enable = 0;                 // ADF4351 专有
    pll_config.band_select_clock_mode_high_enable = 0;
    pll_config.clk_divider_12bit = 0;
    pll_config.clk_divider_mode = 0;

    // R4 用户设置
    pll_config.aux_output_enable = 0;
    pll_config.aux_output_fundamental_enable = 0;
    pll_config.mute_till_lock_enable = 1;
    pll_config.output_power = 1; // 主输出功率档位：0..3
    pll_config.aux_output_power = 0;

    // 2) 先确保上电（CE=1）
    adf4350_out_altvoltage0_powerdown(0);

    // 3) 应用配置并设定上电频率
    adf4350_setup(pll_config);

    // 4) 运行时设置目标频率（例：100 MHz）
    adf4350_out_altvoltage0_frequency(100000000);
}
```

---

## API 说明

- int32_t adf4350_setup(adf4350_init_param init_param)
  - 初始化内部状态，根据 init_param 计算并下装寄存器，设定上电频率

- int32_t adf4350_write(uint32_t data)
  - 低层 32bit 写寄存器（内部已处理分时序与 LE/CE 控制）

- int64_t adf4350_out_altvoltage0_frequency(int64_t Hz)
  - 设置主输出频率（Hz），返回实际设置值（可能与期望有微小偏差）

- int32_t adf4350_out_altvoltage0_frequency_resolution(int32_t Hz)
  - 设置频道间隔（Channel Spacing），影响分数-N分频计算与 MOD 值

- int64_t adf4350_out_altvoltage0_refin_frequency(int64_t Hz)
  - 设置参考输入（Hz），通常等于晶振或外部参考频率（结合倍频/二分频）

- int32_t adf4350_out_altvoltage0_powerdown(int32_t pwd)
  - 0：上电；1：掉电（通过 REG2 的 POWER_DOWN_EN 控制）

---

## 初始化参数详解

结构体 adf4350_init_param（见 adf4351.h）关键字段：
- clkin：参考输入频率（Hz），如 25e6
- channel_spacing：频道间隔（Hz），越小频率步进越细，计算复杂度增
- power_up_frequency：初始化后的上电频点（Hz）
- reference_div_factor：R 分频（1 表示不分频；写入 1..1023）
- reference_doubler_enable：参考倍频 ×2 开关
- reference_div2_enable：参考二分频 ÷2 开关

R2 用户设置：
- phase_detector_polarity_positive_enable：鉴相器正极性
- lock_detect_precision_6ns_enable：锁定检测 6ns 精度
- lock_detect_function_integer_n_enable：整数-N 锁定检测
- charge_pump_current：电荷泵电流（单位 uA，范围约 312~5000，步进 312）
- muxout_select：MUXOUT 功能选择（典型 6=数字锁定检测）
- low_spur_mode_enable：低杂散模式（可能提升杂散，但增大相噪）

R3 用户设置（部分为 ADF4351 专有项）：
- cycle_slip_reduction_enable：CSR
- charge_cancellation_enable：电荷消除
- anti_backlash_3ns_enable：3ns 反冲
- band_select_clock_mode_high_enable：Band Select 高电平模式
- clk_divider_12bit / clk_divider_mode：分频时钟与模式

R4 用户设置：
- aux_output_enable：开关辅输出
- aux_output_fundamental_enable：辅输出是否为基波
- mute_till_lock_enable：未锁定时静音
- output_power / aux_output_power：输出功率档（0..3）

---

## 频率规划与限制

规格（见 adf4351.h 宏定义）：
- 输出频率范围：34.375 MHz ~ 4.4 GHz
- VCO 最小频率：2.2 GHz（低于此频率将通过 RF_DIV 下分频）
- PFD 最大频率：32 MHz
- RMOD（MOD）最大：4095
- R 分频最大：1023
- Band Select 时钟最大：125 kHz

注意：
- 当目标频率低于 2.2 GHz 时，驱动会自动提高 VCO 后再用 RF_DIV 下分（1/2/4/...）
- 若 clkin、R 分频、倍频/二分频 与 channel_spacing 的组合导致 MOD>4095，驱动会尝试提升 channel_spacing
- 实际可达频率会受参考源、环路参数、布局布线、输出匹配等影响

---

## 常见问题

- 上不了锁/频率不对：
  - 确认参考输入 clkin 与倍频/二分频开关设置正确
  - 合理设置 channel_spacing，避免 MOD 超限
  - 检查电荷泵电流、环路滤波器参数是否匹配你的硬件
  - 确认 LE/CE 管脚被正确驱动；MUXOUT 设为 6 可观察数字锁定
- 没有输出：
  - 确认未掉电（adf4350_out_altvoltage0_powerdown(0)）
  - REG4 的 RF_OUT_EN 已在驱动中打开；若需要 Aux OUT，请开启相应开关
- SPI 通讯异常：
  - 检查 SPI 模式（Mode 0）、线序与时序，降低 SPI 频率进行排查

---

## 许可证
本项目遵循本仓库中的 LICENSE 文件。
