# ADF4351 Driver for stm32

> [original project](https://github.com/kb3gtn/STM32_ADF4351)

## Usage

### Pin

SPI1_SCK   -> CLK  
SPI1_MOSI  -> DATA  
ADF4351_LE -> LE  
ADF4351_CE -> CE  

### Initialization

```c
adf4350_init_param pll_config; 

// initialize pll_config structure
pll_config.clkin = 25e6;  // onboard 25MHz Crystal
pll_config.channel_spacing = 100;
pll_config.power_up_frequency = 50e6;
pll_config.reference_div_factor = 1;
pll_config.reference_doubler_enable = 0;
pll_config.reference_div2_enable = 0;
pll_config.phase_detector_polarity_positive_enable = 1;
pll_config.lock_detect_precision_6ns_enable = 1;      // 6 ns
pll_config.lock_detect_function_integer_n_enable = 0; // Fractional pll
pll_config.charge_pump_current = 7;                   // 2.50
pll_config.muxout_select = 6;        // Digital Lock Detect Out
pll_config.low_spur_mode_enable = 0; // higher noise, lower spurs.
pll_config.cycle_slip_reduction_enable = 0;
pll_config.charge_cancellation_enable = 0;
pll_config.anti_backlash_3ns_enable = 0;
pll_config.band_select_clock_mode_high_enable = 0; // low
pll_config.clk_divider_12bit = 0;
pll_config.clk_divider_mode = 0;
pll_config.aux_output_enable = 0;
pll_config.aux_output_fundamental_enable = 0;
pll_config.mute_till_lock_enable = 1;
pll_config.output_power = 1; // +1 dBm
pll_config.aux_output_power = 0;

adf4350_out_altvoltage0_powerdown(0); // power up PLL
adf4350_setup(pll_config);

// set frequency
adf4350_out_altvoltage0_frequency(100000000);
```
