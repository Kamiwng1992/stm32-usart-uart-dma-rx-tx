# Use DMA to receive UART data on STM32 with unknown data length

This repository may give you information about how to read data on UART by using DMA when number of bytes to receive is now known in advance.

In STM32 microcontroller family, U(S)ART reception can work in different modes:

- Polling mode (no DMA, no IRQ): Application must poll for status bits to check if new character has been received and read it fast enough in order to get all bytes
	- PROS
		- Very easy implementation
	- CONS
		- Very easy to loose receive characters
		- Works only in low baudrates
- Interrupt mode (no DMA): UART triggers interrupt and CPU jumps to service routine to handle data reception
	- PROS
		- Most commonly used approach across all applications today
		- Works well in common baudrate, `115200` bauds
	- CONS
		- Interrupt service routing is executed for every received character
		- May stall other tasks in high-performance MCUs with many interrupts
		- May stall operating system when receiving burst of data at a time
- DMA mode: DMA is used to transfer data from USART RX data register to user memory on hardware level. No application interaction is needed at this point except processing received data by application once necessary
	- PROS
		- Transfer from USART peripheral to memory is done on hardware level without CPU interaction
		- Can work very easily with operating systems
	- CONS
		- Number of bytes to transfer must be known in advance by DMA hardware
		- If communication fails, DMA may not notify application about all bytes transferred

This article focuses only on *DMA mode* with unknown data length to receive.

### Important facts about DMA

DMA in STM32 can work in `normal` or `circular` mode. For each mode, it requires number of *elements* to transfer before events are triggered.

- *Normal mode*: In this mode, DMA starts transferring data and when it transfers all elements, it stops.
- *Circular mode*: In this mode, DMA starts with transfer, but when it reaches to the end, it jumps back on top of memory and continues to write

While transfer is active, `2` of many interrupts may be triggered:

- *Half-Transfer complete (HT)* interrupt: Executed when half of elements were transferred by DMA hardware
- *Transfer-Complete (TC)* interrupt: Executed when all elements transferred by DMA hardware
	- When DMA operates in *circular* mode, these interrupts are executed periodically

*Number of elements to transfer by DMA hardware must be written to relevant DMA registers!*

As you can see, we get notification by DMA on *HT* or *TC* events. Imagine application assumes it will receive `20` bytes, but it receives only `14`:
- Application would write `20` to relevant register for number of bytes to receive
- Application would be notfified after first `10` bytes are received (HT event)
- **Application is never notified for the rest of `4` bytes**
    - **Application must solve this case!**

### Important facts about U(S)ART

Most of STM32 series have U(S)ARTs with IDLE line detection. If IDLE line detection is not available, some of them have *Receiver Timeout* feature with programmable delay. If even this is not available, then application may use only *polling modes with DMA*, with examples provided below.

IDLE line detection (or Receiver Timeout) can trigger USART interrupt when receive line is steady without any communication for at least *1* character for reception.
Practicle example: Imagine we received *10* bytes at *115200* bauds. Each byte at *115200* bauds takes about `10us` on UART line, total `100us`. IDLE line interrupt will notify application when it will detect for `1` character inactivity on RX line, meaning after `10us` after last character. Application may react on this event and process data accordingly.

### Connect DMA + USARTs together

Now it is time to use all these features of DMA and USARTs in single application.
If we move to previous example of expecting to receive `20` bytes by application (and actually receiving only `14`), we can now:
- Application would write `20` to relevant register for number of bytes to receive
- Application would be notfified after first `10` bytes are received (HT event)
- Application would be notified after the rest `4` bytes because of USART IDLE line detection (IDLE LINE)

### Final configuration

- Put DMA to `circular` mode to avoid race conditions after DMA transfer completes and before user starts a new transfer
- Set memory length big enough to be able to receive all bytes while processing another.
    - Imagine you receive data at `115200` bauds, bursts of `100` bytes at a time.
    - It is good to set receive buffer to at least `100` bytes unless you can make sure your processing approach is faster than burst of data
    - At `115200` bauds, `100` bytes means `1ms` time

# Examples

All examples were originally developed on `NUCLEO-F413ZH` development board in configuration:

- Developed in TrueSTUDIO and may be directly opened there
- Use of LL drivers
- USART settings
    - `USART3`, `115200` bauds, `1` stop bit, no-parity
    - STM32 TX pin: `PD8`
    - STM32 RX pin: `PD9`
- DMA settings
    - `DMA1`, `STREAM1`, `CHANNEL4`
    - Circular mode
- All examples implement loop-back terminology with polling approach

Examples show different use cases:

### Polling for changes

DMA hardware takes care of transferring received data to memory but application must constantly poll for new changes and read received data fast enough to not get overwritten. Processing of received data is in thread mode
- PROS
	- Easy to implement
	- No interrupts
	- Suitable for devices without *USART IDLE* line detection
- CONS
	- Application must take care of data periodically, fast enough, otherwise data may get overwritten by DMA hardware
	- Harder to get immediate reply when using *USART* based communication

### Polling for changes RTOS

Idea is completely the same as in previous case (*polling only*) but it uses separate thread for data processing
- PROS
	- Easy to implement to RTOS systems, uses only single thread without additional RTOS features
	- No interrupts
	- Data processing always *on-time* with maximum delay given by thread thus with known maximum latency between received character and processing time
	- Suitable for devices without *USART IDLE* line detection
- CONS
	- Uses more memory resources dedicated for separate thread for data processing

### USART Idle line detection + DMA HT&TC interrupts

Similar to `polling` except in this case user gets notification from `3` different sources:
	
- USART idle line detection: Some data was received but now receive line is steady. Interrupt will notify application to process currently received data immediately
	- DMA Half-Transfer (*HT*): DMA hardware reached middle point of received length. Interrupt will norify application to process data immediately. This is to make sure we process data fast enough if we receive burst of data and number of bytes is higher than rolling receive buffer
	- DMA Transfer-Complete (*TC*): Exactly the same like `HT` event, except that it happens at the end of received rolling buffer. Once this event happens, DMA will start receiving from beginning of buffer
- PROS
	- User do not need to check for new changes
	- Relevant interrupts are triggered on which user must immediate react
- CONS
	- Processing of data in this mode is always in interrupt. This may have negative effects on application if there is too much data to process at a time. Doing this may stall CPU and processing of other interrupts

### USART Idle line detection + DMA HT&TC interrupts with RTOS

- The same as `idle_line_irq` type, except it only writes notification to message queue. Data processing is done in separate thread which offloads interrupts for other tasks
- PROS
	- Processing is not in the interrupt, it is in separate thread
- CONS
	- Memory usage for separate thread + message queue (or semaphore)
	- Increases RAM footprint