# ADC_OLED Hardware Configuration

## MCU and Clock

- MCU: `STM32F401CCU6` in the STM32F4 family.
- Package: UFQFPN48.
- HSE is enabled; the `.ioc` records `HSE_VALUE=25000000`.
- PLL values: `PLLM=25`, `PLLN=144`, `PLLP=2`, and `PLLQ=3` in the generated initialization.
- SYSCLK, HCLK, and APB2 are configured for 72 MHz; APB1 is configured for 36 MHz.
- USB clock is recorded as 48 MHz.

The repository does not identify the exact external oscillator component or board model.

## ADC Inputs

ADC1 uses four analog GPIO inputs:

| Rank | ADC channel | GPIO | Sampling |
|---:|---:|---|---|
| 1 | ADC channel 4 | PA4 / ADC1_IN4 | 28 cycles |
| 2 | ADC channel 1 | PA1 / ADC1_IN1 | 28 cycles |
| 3 | ADC channel 2 | PA2 / ADC1_IN2 | 28 cycles |
| 4 | ADC channel 3 | PA3 / ADC1_IN3 | 28 cycles |

ADC resolution is 12-bit, data is right-aligned, scan mode is enabled, continuous mode is disabled, and the external trigger is a rising TIM3 TRGO event.

## DMA Resource

- Controller/stream: DMA2 Stream 0.
- Channel: DMA channel 0.
- Direction: peripheral to memory.
- Peripheral increment: disabled.
- Memory increment: enabled.
- Peripheral and memory alignment: halfword.
- Mode: circular.
- Priority: high.
- FIFO mode: disabled.
- Interrupt: `DMA2_Stream0_IRQn`, priority 5, subpriority 0.

The destination buffer is `uint16_t adc_values[4]` in `main.c`.

## Timer Trigger

TIM3 uses the internal clock and emits `TIM_TRGO_UPDATE`. The generated configuration uses prescaler `7199` and period `9`. At the configured 72 MHz timer-domain clock, the calculated update frequency is 1 kHz. TIM3 is started by `ADC_task` after the ADC DMA transfer is started.

## OLED Interface

- Peripheral: I2C1.
- Pins: PB6 = I2C1_SCL and PB7 = I2C1_SDA.
- I2C speed: 400000 Hz.
- Addressing: 7-bit mode in the I2C configuration.
- Driver address constant: `SSD1306_I2C_ADDR=0x78`.
- Display geometry: 128x64 pixels.
- Driver API: local `ssd1306.c/.h` using `HAL_I2C_Mem_Write()`.

The repository does not specify the physical OLED module wiring, pull-up values, or display vendor beyond the SSD1306-compatible driver configuration.

## USB

USB OTG FS is configured in device-only mode with the Full-Speed CDC class:

- PA11 = USB_OTG_FS_DM.
- PA12 = USB_OTG_FS_DP.
- USB device initialization is performed by `LoggerTask`.
- The application transmits formatted ADC log lines through `CDC_Transmit_FS()`.
- The receive callback re-arms the receive buffer but does not interpret received bytes.

## GPIO

- PC13 is configured as a low-speed push-pull output with no pull resistor and reset initial level.
- The `Led` task toggles PC13 every 500 RTOS ticks.

## Other Interrupt Resources

- USB OTG FS interrupt calls `HAL_PCD_IRQHandler()`.
- TIM11 is selected as the HAL timebase and its period callback calls `HAL_IncTick()`.
- PendSV is configured at priority 15 for the RTOS port.
- SysTick is configured at priority 15 in the CubeMX project settings.
