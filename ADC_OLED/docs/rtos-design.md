# ADC_OLED RTOS Design

## Task Design

The project uses CMSIS-RTOS2 APIs backed by FreeRTOS kernel 10.3.1. Threads are dynamically created with the generated attributes in `Core/Src/main.c`.

| Task | Priority | Stack size | Blocking/timing behavior |
|---|---:|---:|---|
| `defaultTask` | 24 (`osPriorityNormal`) | 512 bytes | Exits immediately. |
| `ADC_task` | 32 (`osPriorityAboveNormal`) | 2048 bytes | Blocks indefinitely on `AdcDoneSem`, then calls `osDelay(1)`. |
| `Display` | 24 (`osPriorityNormal`) | 3072 bytes | Blocks indefinitely on `AdcQueue`, then calls `osDelay(1)`. |
| `LoggerTask` | 24 (`osPriorityNormal`) | 3072 bytes | Blocks indefinitely on `LogQueue`; retries busy CDC transmission and calls `osDelay(1)`. |
| `Led` | 8 (`osPriorityLow`) | 2048 bytes | Toggles PC13 and uses `osDelayUntil()` with a 500-tick increment, followed by `osDelay(1)`. |

The RTOS tick is configured at 1000 Hz. The application does not provide runtime scheduling measurements or deadline analysis.

## Scheduling

`configUSE_PREEMPTION` is enabled. The ADC task has a higher CMSIS priority than the display and logger tasks, while the LED task has a lower priority. Queue and semaphore waits use `osWaitForever`, so the consuming tasks sleep when no work is available.

The ADC task starts TIM3 and ADC DMA once, then waits for the completion semaphore. Display and logging work is performed outside the ADC callback.

## Queues

| Queue | Capacity | Element type | Writer | Reader |
|---|---:|---|---|---|
| `AdcQueue` | 4 | `AdcMessage_t` containing four `uint16_t` values | `startADC` | `StartDisplay` |
| `LogQueue` | 8 | `LogMessage_t` containing `char text[128]` | `startADC`, `AppLog` | `StartLogger` |

The source calls `osMessageQueuePut()` with zero timeout. The return status is not checked, so a full queue can cause a message to be dropped without an application-level diagnostic.

## Semaphores

`AdcDoneSem` is created with maximum count 1 and initial count 1. `HAL_ADC_ConvCpltCallback()` releases it when the callback is associated with ADC1. `startADC()` acquires it with `osWaitForever` before copying the four DMA results.

The source does not use a semaphore for USB completion; CDC busy status is handled by polling inside `LoggerTask` while holding `UsbMutex`.

## Mutexes

- `OledMutex` surrounds SSD1306 initialization and the fill/write/update sequence.
- `UsbMutex` surrounds the CDC transmit retry loop.

The code creates both mutexes dynamically before starting the scheduler. It does not check the return value from `osMutexNew()` or from the mutex operations beyond testing for `osOK` on acquisition.

## Critical Sections

No application-level `taskENTER_CRITICAL()`, `taskEXIT_CRITICAL()`, or equivalent critical-section block appears in the application source. Kernel interrupt masking is configured by the FreeRTOS port and HAL.

## ISR-to-Task Communication

`DMA2_Stream0_IRQHandler()` calls `HAL_DMA_IRQHandler(&hdma_adc1)`. The HAL dispatches the ADC conversion-complete callback, which calls `osSemaphoreRelease(AdcDoneSemHandle)`. DMA2 Stream 0 is configured at NVIC priority 5, matching the configured FreeRTOS maximum syscall interrupt priority boundary. The callback does not use an explicit `FromISR` API in application code; it calls the CMSIS-RTOS2 semaphore API.

USB and timer interrupt handlers call the corresponding HAL handlers. The USB CDC callbacks do not post an RTOS object in the current application.

## Timing Considerations

- TIM3 is configured with prescaler `7199`, period `9`, and update TRGO. Using the configured 72 MHz timer clock, this calculates to a 1 kHz trigger rate.
- `Adc_task`, `Display`, and `LoggerTask` each add `osDelay(1)` after their main operation.
- `LoggerTask` can hold `UsbMutex` while retrying for up to 500 ms if CDC remains busy.
- `Led` uses a 500-tick increment, equivalent to 500 ms at the configured tick rate.
- No measured queue latency, CPU load, stack high-water mark, or end-to-end timing is included in the repository.

The circular DMA and queue capacities should be reviewed if processing cannot keep up with the trigger rate. Return values from queue operations should also be handled if loss detection is required.
