# Hardware Overview

## MCU

- Part number: STM32F401CCUx
- Family: STM32F4
- Package: UFQFPN48
- System clock: 72 MHz
- External oscillator: 25 MHz HSE

## Peripherals

### ADC

- ADC peripheral: ADC1
- ADC channels configured in scan mode: 4
- Channel mapping:
  - PA1 -> ADC1_IN1
  - PA2 -> ADC1_IN2
  - PA3 -> ADC1_IN3
  - PA4 -> ADC1_IN4
- Resolution: 12-bit
- Sampling time: 28 cycles for each rank
- Conversion mode: continuous
- Data alignment: right

### DMA

- DMA instance: DMA2 Stream0
- Data direction: peripheral to memory
- Mode: circular
- Alignment: half-word
- Priority: high

### I2C OLED interface

- Peripheral: I2C1
- Pins: PB6 (SCL), PB7 (SDA)
- Speed: 400 kHz
- Display: SSD1306 128x64
- Address: 0x78

### USB

- Peripheral: USB OTG FS
- Pins: PA11 (DM), PA12 (DP)
- Device class: CDC
- Function: USB virtual COM port

### GPIO

- PC13: LED output, toggled by `task2`

## Clock configuration

The system clock is configured as follows in the generated code:

- HSE enabled at 25 MHz
- PLL enabled with source = HSE
- PLLM = 25
- PLLN = 144
- PLLP = DIV2
- PLLQ = 3
- SYSCLK source = PLLCLK
- SYSCLK frequency = 72 MHz
- APB1 = 36 MHz
- APB2 = 72 MHz

## Interfaces and signal routing

- ADC1 inputs routed to PA1, PA2, PA3, PA4
- I2C1 routed to PB6 and PB7
- USB OTG FS routed to PA11 and PA12
- LED output routed to PC13

The pin configuration and peripheral mapping are recorded in the project `.ioc` file and reflected in the generated firmware initialization code.
