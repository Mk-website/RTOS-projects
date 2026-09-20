# Architecture

## System architecture

This project is built around a STM32F401CCU6 MCU running FreeRTOS and CMSIS-RTOS v2 tasks. The architecture combines ADC acquisition, DMA transfer, USB device communication, and a display subsystem, all coordinated by the RTOS scheduler and generated HAL drivers.

The main system elements are:

- CPU: STM32F401CCU6
- Clock: 25 MHz HSE with PLL generating 72 MHz SYSCLK
- ADC: ADC1 with four regular conversions in scan mode
- DMA: DMA2 Stream0 for ADC peripheral-to-memory transfer
- Display: SSD1306 OLED over I2C1
- USB: USB OTG FS CDC device
- GPIO: PC13 for LED output

## Task architecture

The Cube-generated task list defines three threads:

1. `defaultTask` — USB and ADC sample publisher
2. `Task1` — OLED display updater
3. `Task2` — LED toggler

The system uses task-level separation for these functions, but the current source does not create queues, semaphores, or mutexes.

## ADC DMA data path

The ADC data path is:

- `ADC1` samples configured channels on PA1, PA2, PA3, and PA4
- DMA2 Stream0 transfers the conversion results into `adc_values[4]`
- `HAL_ADC_ConvCpltCallback` sets `dma_complete = 1`
- `defaultTask` reads the buffer and sends formatted strings to USB CDC

## OLED data path

The OLED data path is:

- `Task1` calls `ssd1306_Init(&hi2c1)`
- `ssd1306_WriteString()` writes text into the software frame buffer
- `ssd1306_UpdateScreen(&hi2c1)` transfers the page data to the SSD1306 over I2C1

The SSD1306 display runs at address `0x78` and uses the 128x64 pixel format.

## USB CDC data path

The USB CDC path is:

- `StartDefaultTask` calls `MX_USB_DEVICE_Init()`
- `print()` calls `CDC_Transmit_FS()`
- `USBD_CDC_SetTxBuffer()` and `USBD_CDC_TransmitPacket()` move the payload to the USB CDC interface
- The payload is a formatted string containing ADC values

## RTOS synchronization

The project does not use explicit RTOS synchronization primitives in the source code. The main synchronizing mechanism is the global flag `dma_complete`, set in the ADC conversion complete callback and checked in the default task.

## Resource ownership

The current design is simple and maintains task separation by convention rather than enforced ownership rules:

- `defaultTask` owns the ADC DMA data flow and USB CDC output
- `Task1` owns the OLED display updates
- `Task2` owns GPIO PC13 toggling

No mutexes or semaphores are used to protect these resources in the application code.
