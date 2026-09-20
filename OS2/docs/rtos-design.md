# RTOS Design

## Task design

The project is generated as a CMSIS-RTOS v2 application and defines three tasks in the `.ioc` file:

- `defaultTask` — normal priority, 128 words stack
- `Task1` — low priority, 128 words stack
- `Task2` — low priority, 128 words stack

These tasks are created in `main.c` with `osThreadNew()` and configured through `osThreadAttr_t` objects.

## Scheduling

The scheduler is started via `osKernelStart()`, and the tasks use `osDelay()` for periodic execution. The code demonstrates cooperative timing, but not advanced task synchronization. The use of `osDelay(1)` in the ADC publisher loop and `osDelay(500)` in the OLED and LED tasks is based on the generated firmware and is not a custom RTOS timing framework.

## Priorities

The priority values in the source are:

- `defaultTask`: `osPriorityNormal` = 24
- `Task1`: `osPriorityLow` = 8
- `Task2`: `osPriorityLow` = 8

The CubeMX `.ioc` file confirms the same values: `defaultTask,24,128` and `Task1,8,128;Task2,8,128`.

## Queues

No FreeRTOS queue objects are created in the project source. There are no calls to:

- `xQueueCreate()`
- `osMessageQueueNew()`
- `osMailQCreate()`

The generated configuration enables queue-related features only at the kernel level; it does not instantiate a user queue in the application.

## Semaphores

No semaphore creation or acquisition calls are found. There are no references to:

- `xSemaphoreCreate()`
- `osSemaphoreNew()`
- `osSemaphoreAcquire()`

The FreeRTOS configuration enables counting semaphores and mutex support, but the application does not use them.

## Mutexes

No mutex objects are created in the source code. There are no `osMutexNew()` or `xSemaphoreCreateMutex()` calls.

## Critical sections

The project does not contain application-specific critical sections implemented through `taskENTER_CRITICAL()` or `taskEXIT_CRITICAL()`. The generated `Error_Handler()` disables interrupts and enters an infinite loop in the failure case.

## ISR-to-task communication

ADC completion and error events are communicated by interrupt callbacks:

- `HAL_ADC_ConvCpltCallback()` sets `dma_complete = 1`
- `HAL_ADC_ErrorCallback()` sets `dma_error = 1`

These flags are then checked by the main RTOS task code. The project does not use task notifications or queue-based ISR communication.

## Timing considerations

The scheduler timing is controlled by the generated System Tick and FreeRTOS tick configuration. The tasks and the USB logging loop rely on short delays (`osDelay(1)`, `osDelay(500)`), which are enough for the simple demonstration but do not implement a more advanced event-driven scheduling strategy.

## Summary

This project demonstrates an RTOS-based embedded design with three tasks and simple timing-based coordination. It is a clear educational example of how a task can manage ADC sampling and USB output while a second task refreshes an OLED display and a third toggles an LED.
