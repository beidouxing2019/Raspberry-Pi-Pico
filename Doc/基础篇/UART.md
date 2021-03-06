## UART

USART是一个全双工通用同步/异步串行收发模块，该接口是一个高度灵活的串行通信设备。USART收发模块一般分为三大部分：[时钟发生器](https://baike.baidu.com/item/时钟发生器/9827742)、数据发送器和接收器。控制[寄存器](https://baike.baidu.com/item/寄存器)为所有的模块共享。

### MicroPython

| **Function    | **Default** |
| ------------- | ----------- |
| UART_BAUDRATE | 115,200     |
| UART_BITS     | 8           |
| UART_STOP     | 1           |
| UART0_TX      | Pin 0       |
| UART0_RX      | Pin 1       |
| UART1_TX      | Pin 4       |
| UART1_RX      | Pin 5       |

```python
from machine import Pin
from rp2 import PIO, StateMachine, asm_pio

UART_BAUD = 115200
PIN_BASE = 10
NUM_UARTS = 8


@asm_pio(sideset_init=PIO.OUT_HIGH, out_init=PIO.OUT_HIGH, out_shiftdir=PIO.SHIFT_RIGHT)
def uart_tx():
    # Block with TX deasserted until data available
    pull()
    # Initialise bit counter, assert start bit for 8 cycles
    set(x, 7)  .side(0)       [7]
    # Shift out 8 data bits, 8 execution cycles per bit
    label("bitloop")
    out(pins, 1)              [6]
    jmp(x_dec, "bitloop")
    # Assert stop bit for 8 cycles total (incl 1 for pull())
    nop()      .side(1)       [6]


# Now we add 8 UART TXs, on pins 10 to 17. Use the same baud rate for all of them.
uarts = []
for i in range(NUM_UARTS):
    sm = StateMachine(
        i, uart_tx, freq=8 * UART_BAUD, sideset_base=Pin(PIN_BASE + i), out_base=Pin(PIN_BASE + i)
    )
    sm.active(1)
    uarts.append(sm)

# We can print characters from each UART by pushing them to the TX FIFO
def pio_uart_print(sm, s):
    for c in s:
        sm.put(ord(c))


# Print a different message from each UART
for i, u in enumerate(uarts):
    pio_uart_print(u, "Hello from UART {}!\n".format(i))
```

### C/C++

以下为测试demo：

```c
#include "pico/stdlib.h"
#include "hardware/uart.h"
#include "hardware/irq.h"


/// \tag::uart_advanced[]

#define UART_ID uart0
#define BAUD_RATE 115200
#define DATA_BITS 8
#define STOP_BITS 1
#define PARITY    UART_PARITY_NONE

// We are using pins 0 and 1, but see the GPIO function select table in the
// datasheet for information on which other pins can be used.
#define UART_TX_PIN 0
#define UART_RX_PIN 1

static int chars_rxed = 0;

// RX interrupt handler
void on_uart_rx() {
    while (uart_is_readable(UART_ID)) {
        uint8_t ch = uart_getc(UART_ID);
        // Can we send it back?
        if (uart_is_writable(UART_ID)) {
            // Change it slightly first!
            ch++;
            uart_putc(UART_ID, ch);
        }
        chars_rxed++;
    }
}

int main() {
    // Set up our UART with a basic baud rate.
    uart_init(UART_ID, 2400);

    // Set the TX and RX pins by using the function select on the GPIO
    // Set datasheet for more information on function select
    gpio_set_function(UART_TX_PIN, GPIO_FUNC_UART);
    gpio_set_function(UART_RX_PIN, GPIO_FUNC_UART);

    // Actually, we want a different speed
    // The call will return the actual baud rate selected, which will be as close as
    // possible to that requested
    int actual = uart_set_baudrate(UART_ID, BAUD_RATE);

    // Set UART flow control CTS/RTS, we don't want these, so turn them off
    uart_set_hw_flow(UART_ID, false, false);

    // Set our data format
    uart_set_format(UART_ID, DATA_BITS, STOP_BITS, PARITY);

    // Turn off FIFO's - we want to do this character by character
    uart_set_fifo_enabled(UART_ID, false);

    // Set up a RX interrupt
    // We need to set up the handler first
    // Select correct interrupt for the UART we are using
    int UART_IRQ = UART_ID == uart0 ? UART0_IRQ : UART1_IRQ;

    // And set up and enable the interrupt handlers
    irq_set_exclusive_handler(UART_IRQ, on_uart_rx);
    irq_set_enabled(UART_IRQ, true);

    // Now enable the UART to send interrupts - RX only
    uart_set_irq_enables(UART_ID, true, false);

    // OK, all set up.
    // Lets send a basic string out, and then run a loop and wait for RX interrupts
    // The handler will count them, but also reflect the incoming data back with a slight change!
    uart_puts(UART_ID, "\nHello, uart interrupts\n");

    while (1)
        tight_loop_contents();
}
```

CMakeLists.txt:

```cmake
add_executable(uart_advanced
        uart_advanced.c
        )

# Pull in our pico_stdlib which pulls in commonly used features
target_link_libraries(uart_advanced pico_stdlib hardware_uart)

# create map/bin/hex file etc.
pico_add_extra_outputs(uart_advanced)

# add url via pico_set_program_url
example_auto_set_url(uart_advanced)
```

