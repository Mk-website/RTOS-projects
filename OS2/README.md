# STM32F401CCU6 RTOS ADC DMA + OLED + USB CDC Demo

## Overview

This project demonstrates a FreeRTOS-based STM32F401CCU6 application that combines ADC sampling with DMA, a 128x64 SSD1306 OLED display, and USB CDC serial communication. The firmware is generated from STM32CubeIDE/STM32CubeMX and is structured around multiple CMSIS-RTOS v2 tasks that run concurrently on the microcontroller.

The application is useful as an embedded RTOS example because it shows how independent tasks can cooperate around peripheral resources such as the ADC, USB device interface, and I2C OLED display while preserving a simple, readable task structure.

## Key Features

- STM32F401CCU6 MCU with STM32CubeIDE project generation
- FreeRTOS task scheduling via CMSIS-RTOS v2
- ADC1 scan conversion with DMA transfer to a 4-sample buffer
- USB OTG FS CDC serial output
- SSD1306 OLED display driven over I2C1
- LED toggle on PC13 from a RTOS task
- Error handling through the project-wide `Error_Handler()` routine

## Hardware

- MCU: STM32F401CCU6 (part number from the CubeMX configuration: `STM32F401CCUx`)
- Core clock: 72 MHz system clock configured from external 25 MHz HSE with PLL
- ADC: ADC1, 4-channel scan sequence
  - Channel 4 on PA4 (`ADC_CHANNEL_4`)
  - Channel 1 on PA1 (`ADC_CHANNEL_1`)
  - Channel 2 on PA2 (`ADC_CHANNEL_2`)
  - Channel 3 on PA3 (`ADC_CHANNEL_3`)
- DMA: DMA2 Stream0 configured for ADC1 peripheral-to-memory transfer
- OLED display: SSD1306, 128x64, connected through I2C1
  - SCL: PB6
  - SDA: PB7
  - I2C bus speed: 400 kHz
- USB: USB OTG FS device interface
  - DM: PA11
  - DP: PA12
  - CDC virtual COM port class used by the project
- User LED: PC13 driven as GPIO output

## Software Stack

- STM32CubeIDE project generated with STM32CubeMX
- STM32 HAL drivers for ADC, GPIO, DMA, I2C, USB, and system clock configuration
- FreeRTOS kernel version 10.3.1 as configured in `FreeRTOSConfig.h`
- CMSIS-RTOS v2 API used by the Cube-generated task definitions
- C programming language
- ST USB Device Library for CDC class support
- SSD1306 display driver implemented in `ssd1306.c` and `ssd1306.h`
- Custom font definitions in `fonts.c` and `fonts.h`

## RTOS Architecture

The project creates three CMSIS-RTOS v2 threads in the generated `main.c` code:

- `defaultTask` runs `StartDefaultTask`
- `Task1` runs `task1`
- `Task2` runs `task2`

The application uses the RTOS for task scheduling and cooperative timing, but no application-level queues, semaphores, mutexes, or task notifications are instantiated in the current source. The configuration file enables mutex and counting semaphore support, but the application does not create or use these objects.

```mermaid
flowchart LR
    A[defaultTask\nStartDefaultTask] --> B[USB Device Init]
    A --> C[ADC1 + DMA2 Stream0]
    C --> D[adc_values[4]]
    D --> E[CDC_Transmit_FS\nUSB CDC output]
    F[Task1\ntask1] --> G[SSD1306 Init]
    G --> H[OLED Update\nI2C1]
    I[Task2\ntask2] --> J[GPIOC PC13 toggle]
    K[HAL_ADC_ConvCpltCallback] --> L[dma_complete flag]
    L --> A
```

## Tasks

| Task | Priority | Stack Size | Responsibility |
|------|----------|------------|----------------|
| `defaultTask` / `StartDefaultTask` | `osPriorityNormal` (24) | 128 words = 512 bytes | Initializes USB CDC, starts ADC DMA, formats ADC readings, sends them over CDC, loops with `osDelay(1)` |
| `Task1` / `task1` | `osPriorityLow` (8) | 128 words = 512 bytes | Initializes the SSD1306 OLED, writes a startup message, updates a counter on the display every 500 ms |
| `Task2` / `task2` | `osPriorityLow` (8) | 128 words = 512 bytes | Toggles GPIO PC13 every 500 ms |

## Inter-Task Communication

No RTOS message queues, mutexes, counting semaphores, or task notifications are created in the current application. The project therefore relies on shared global state and peripheral access patterns instead of explicit RTOS synchronization objects.

| Mechanism | Purpose | Producer | Consumer/User |
|-----------|---------|----------|---------------|
| None found | No FreeRTOS queue is created | Not applicable | Not applicable |
| None found | No semaphore instance is created | Not applicable | Not applicable |
| None found | No mutex instance is created | Not applicable | Not applicable |
| Global flag: `dma_complete` | Indicates ADC DMA conversion complete callback fired | `HAL_ADC_ConvCpltCallback` | `StartDefaultTask` |
| Global buffer: `adc_values[4]` | Stores ADC conversion results from DMA | ADC DMA transfer | `StartDefaultTask` |

## ADC + DMA

The ADC configuration is controlled in `MX_ADC1_Init()` and the associated `SystemClock_Config()` routine.

- ADC peripheral: `ADC1`
- Resolution: `ADC_RESOLUTION_12B`
- Scan mode: enabled
- Continuous conversion mode: enabled
- DMA continuous requests: enabled
- Data alignment: right-aligned
- Trigger source: software start (`ADC_SOFTWARE_START`)
- Number of regular conversions: 4
- Sampling time: `ADC_SAMPLETIME_28CYCLES` for each rank
- External trigger: none (`ADC_EXTERNALTRIGCONVEDGE_NONE`)

The conversion sequence is configured as:

1. `ADC_CHANNEL_4` rank 1
2. `ADC_CHANNEL_1` rank 2
3. `ADC_CHANNEL_2` rank 3
4. `ADC_CHANNEL_3` rank 4

DMA is configured for ADC1 with:

- DMA controller: `DMA2_Stream0`
- Direction: peripheral-to-memory
- Data alignment: half-word for peripheral and memory
- Memory increment: enabled
- Peripheral increment: disabled
- Mode: circular
- Priority: high

The application starts the ADC-DMA transfer with:

```c
if (HAL_ADC_Start_DMA(&hadc1, (uint32_t *)adc_values, 4) != HAL_OK) {
    print("ADC DMA Start Failed\r\n");
    Error_Handler();
}
```

The DMA result buffer is `uint16_t adc_values[4];`. The callback functions are:

```c
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
   if (hadc->Instance == ADC1) dma_complete = 1;
}

void HAL_ADC_ErrorCallback(ADC_HandleTypeDef *hadc)
{
   if (hadc->Instance == ADC1) dma_error = 1;
}
```

The default task then reads the buffer and sends the sampled values over USB CDC.

## OLED Display

The project uses a custom SSD1306 driver implemented in `ssd1306.c` and `ssd1306.h`.

- Driver/controller: SSD1306
- Interface: I2C
- Bus: I2C1
- Address: `0x78` (`SSD1306_I2C_ADDR`)
- Resolution: 128 x 64 pixels
- I2C speed: 400 kHz (`I2C_Fast`)

The start-up sequence in `task1()` calls:

```c
ssd1306_Init(&hi2c1);
ssd1306_Fill(Black);
ssd1306_SetCursor(0, 10);
ssd1306_WriteString("Manjit ", Font_11x18, White);
ssd1306_UpdateScreen(&hi2c1);
```

The driver writes command bytes and page data to the display using `HAL_I2C_Mem_Write()`. The display buffer is updated with the SSD1306 page-addressing sequence and then transferred to the panel in 8 page segments.

## USB CDC

The project is configured for a USB Device CDC interface using the ST USB device library. The relevant CubeMX settings include:

- `USB_DEVICE` class: `CDC`
- USB mode: `Device_Only`
- USB OTG FS: `USB_OTG_FS`
- Virtual mode: `CDC_FS`

The application calls `MX_USB_DEVICE_Init()` from `StartDefaultTask()` before starting ADC DMA. The code then uses `CDC_Transmit_FS()` to send data packets over the USB virtual COM port. The helper function `print()` checks the return value and waits with `osDelay(1)` while the USB interface reports `USBD_BUSY`.

The transmitted content consists of formatted ADC sample strings, such as:

```c
snprintf(msg, sizeof(msg), "ADC values : %3u | %3u | %3u | %3u | \r\n",
         adc_values[0], adc_values[1], adc_values[2], adc_values[3]);
print(msg);
```

No USB receive callback logic is implemented beyond the default CDC receive handler generated by STM32CubeMX.

## System Data Flow

```mermaid
flowchart LR
    A[ADC1 channels\nPA1 PA2 PA3 PA4] --> B[DMA2 Stream0\nperipheral-to-memory]
    B --> C[adc_values[4]]
    C --> D[StartDefaultTask]
    D --> E[CDC_Transmit_FS]
    E --> F[USB CDC host]
    D --> G[HAL_ADC_ConvCpltCallback]
    G --> H[dma_complete flag]
    I[Task1] --> J[SSD1306 I2C driver]
    J --> K[128x64 OLED]
```

## Project Structure

Important files and folders in this project include:

- `Core/Src/main.c` — main firmware logic, task creation, ADC setup, USB init, and task bodies
- `Core/Inc/FreeRTOSConfig.h` — FreeRTOS configuration defines, including kernel timing and CMSIS-RTOS V2 settings
- `Core/Src/ssd1306.c` — SSD1306 driver implementation
- `Core/Inc/ssd1306.h` — SSD1306 register and buffer definitions
- `USB_DEVICE/App/usbd_cdc_if.c` — CDC transmit implementation and USB class interface
- `USB_DEVICE/App/usb_device.c` — USB device initialization
- `OS2.ioc` — CubeMX configuration file capturing the pinout, clocks, peripherals, and RTOS task list
- `Debug/` — generated build artifacts and dependency files

## Build and Flash

This project is intended for STM32CubeIDE.

1. Open the folder in STM32CubeIDE.
2. Import or open the existing project generated from `OS2.ioc`.
3. Build the project using the Debug configuration.
4. Connect the STM32F401 board with an ST-LINK or compatible debugger.
5. Flash the firmware from the IDE.
6. Reset the MCU and verify the USB CDC output and display behavior.

The project also contains generated Eclipse project metadata such as `.project`, `.cproject`, and `.launch` files, so the project is ready to be opened directly in CubeIDE without re-generation.

## Configuration

Important CubeMX configuration points that are reflected in the source code:

- External crystal: HSE ON at 25 MHz
- PLL configuration yields 72 MHz system clock
- ADC1 in scan and continuous mode with DMA requests enabled
- DMA2 Stream0 configured to transfer 4 half-word values
- I2C1 configured in fast mode at 400 kHz
- USB OTG FS set to CDC device mode
- GPIO PC13 configured as output LED
- FreeRTOS task list defined in the `.ioc` file:
  - `defaultTask, 24, 128`
  - `Task1, 8, 128`
  - `Task2, 8, 128`

## RTOS Concepts Demonstrated

This project demonstrates the practical use of:

- Task scheduling with three concurrently running tasks
- Periodic timing through `osDelay()` calls
- ADC DMA as a peripheral transfer mechanism independent of the CPU core loop
- USB CDC transmission as an output path from task code
- Peripheral sharing at a high level through task separation, though explicit RTOS synchronization primitives are not used in the current source
- Interrupt-driven ADC completion and error callbacks

The current source does not demonstrate application-level queue, semaphore, or mutex usage.

## Learning Outcomes

An embedded engineer can learn from this project how to:

- Configure a Cortex-M4 microcontroller for FreeRTOS task scheduling
- Integrate ADC with DMA for continuous multi-channel conversion
- Use USB CDC as a simple debug and telemetry interface
- Drive an SSD1306 display over I2C with a custom driver
- Structure a simple multithreaded embedded application around peripheral ownership and timing
- Interpret CubeMX-generated code and map generated settings to actual runtime behavior

## Possible Improvements

These are future improvement ideas that are not implemented in the current source code:

- Add RTOS queues to pass ADC results from the sampling task to a display or telemetry task
- Add a semaphore or mutex around shared debug/USB output paths if multiple tasks transmit concurrently
- Add explicit state/error handling for OLED or USB failures
- Introduce a dedicated telemetry task and a display task with cleaner separation of responsibilities
- Add filtering and calibration for ADC readings before display or transmission
- Add watchdog and fault monitoring for USB or I2C recoverability

## Author

The project author recorded in the repository metadata is: Manjit kumar

## License

This project is distributed under the MIT License. See the LICENSE file in this directory for details.
