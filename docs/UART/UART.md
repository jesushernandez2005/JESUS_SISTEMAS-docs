---
title: "Two-Way UART Communication Between Two Picos (RP2350)"
date: 2026-10-28
tags: [embedded-systems, raspberry-pi-pico-2, uart, serial-communication, gpio, c]
---

# Two-Way UART Communication Between Two Picos (RP2350)

## Objective

Connect two Pico 2 W boards through UART so they control each other: pressing a button on one board turns on the matching LED on the other one. It works in both directions and both boards run the same program.

## Materials

- 2 × Raspberry Pi Pico 2 W
- 6 × LED + 6 × 330 Ω resistor (3 per board)
- 6 × push-button (3 per board)
- 2 × protoboard, jumper wires, 2 × micro-USB cable

## Circuit Diagram

Each board:

```
GPIO2 → LED A → 330Ω resistor → GND
GPIO3 → LED B → 330Ω resistor → GND
GPIO4 → LED C → 330Ω resistor → GND
GPIO5 → Button A → GND   (internal pull-up)
GPIO6 → Button B → GND   (internal pull-up)
GPIO7 → Button C → GND   (internal pull-up)
```

Between the boards:

```
Pico 1 GPIO0 (TX) → Pico 2 GPIO1 (RX)
Pico 1 GPIO1 (RX) ← Pico 2 GPIO0 (TX)
Pico 1 GND        ↔ Pico 2 GND
```

- TX and RX have to be **crossed**: what one board sends has to arrive at the other one's RX.
- **Common GND** is required, because without the same reference the receiver can't tell a `1` from a `0`.

## Source Code

```c
#include "pico/stdlib.h"
#include "hardware/uart.h"
#include <stdio.h>

#define UART_ID uart0
#define BAUD_RATE 115200
#define TX_PIN 0
#define RX_PIN 1
#define Button_A 5
#define Button_B 6
#define Button_C 7
#define Led_A 2
#define Led_B 3
#define Led_C 4

int main() {
    stdio_init_all();

    gpio_init(Button_A);
    gpio_init(Button_B);
    gpio_init(Button_C);
    gpio_init(Led_A);
    gpio_init(Led_B);
    gpio_init(Led_C);

    gpio_set_dir(Led_A, 1);
    gpio_set_dir(Led_B, 1);
    gpio_set_dir(Led_C, 1);
    gpio_set_dir(Button_A, 0);
    gpio_set_dir(Button_B, 0);
    gpio_set_dir(Button_C, 0);

    gpio_pull_up(Button_A);
    gpio_pull_up(Button_B);
    gpio_pull_up(Button_C);

    gpio_set_function(TX_PIN, GPIO_FUNC_UART);
    gpio_set_function(RX_PIN, GPIO_FUNC_UART);

    uart_init(UART_ID, BAUD_RATE);
    uart_set_format(UART_ID, 8, 1, UART_PARITY_NONE);

    printf("[MAIN] Sistema iniciado...\n");

    int flag_a = 0;
    int flag_b = 0;
    int flag_c = 0;

    while (true) {
        // --- BOTON A ---
        if (gpio_get(Button_A) == 0 && flag_a == 0) {
            uart_putc(UART_ID, 'A');  // Se presionó
            flag_a = 1;
        } else if (gpio_get(Button_A) == 1 && flag_a == 1) {
            uart_putc(UART_ID, 'a');  // Se soltó
            flag_a = 0;
        }

        // --- BOTON B ---
        if (gpio_get(Button_B) == 0 && flag_b == 0) {
            uart_putc(UART_ID, 'B');  // Se presionó
            flag_b = 1;
        } else if (gpio_get(Button_B) == 1 && flag_b == 1) {
            uart_putc(UART_ID, 'b');  // Se soltó
            flag_b = 0;
        }

        // --- BOTON C ---
        if (gpio_get(Button_C) == 0 && flag_c == 0) {
            uart_putc(UART_ID, 'C');  // Se presionó
            flag_c = 1;
        } else if (gpio_get(Button_C) == 1 && flag_c == 1) {
            uart_putc(UART_ID, 'c');  // Se soltó
            flag_c = 0;
        }

        // --- RECEPCION UART ---
        while (uart_is_readable(UART_ID)) {
            char c = uart_getc(UART_ID);
 
            // Control Led A
            if (c == 'A') {
                gpio_put(Led_A, 1);
            } else if (c == 'a') {
                gpio_put(Led_A, 0);
            }
 
            // Control Led B
            if (c == 'B') {
                gpio_put(Led_B, 1);
            } else if (c == 'b') {
                gpio_put(Led_B, 0);
            }
 
            // Control Led C
            if (c == 'C') {
                gpio_put(Led_C, 1);
            } else if (c == 'c') {
                gpio_put(Led_C, 0);
            }
        }
 
        sleep_ms(10);
    }
}
```

## How it works

### UART setup

- UART0 on GPIO0/GPIO1, 115200 baud, 8N1.
- Both boards need **the same settings**, because UART has no clock line: each receiver uses the baud rate to know when to read each bit.

### Protocol

Uppercase = pressed / LED on, lowercase = released / LED off:

| Character | Sent when… | Effect on the other board |
|---|---|---|
| `A` / `a` | Button A pressed / released | LED A on / off |
| `B` / `b` | Button B pressed / released | LED B on / off |
| `C` / `c` | Button C pressed / released | LED C on / off |

### Sending

- Each button has a flag (`flag_a`, `flag_b`, `flag_c`), and a character is only sent when the state **changes**.
- Without the flag it would send `'A'` every 10 ms while the button is held.

### Receiving

- `uart_is_readable()` checks if anything arrived, and the `while` reads everything in the FIFO, so nothing is lost if two bytes come in together.
- Each character turns its LED on or off, and anything else is ignored.

Since TX and RX are separate lines, both boards send and receive at the same time (**full-duplex**), so no master/slave is needed.

## Results

- A button on one board lights the matching LED on the other one, for exactly as long as it's held.
- It works both ways at the same time, and with several buttons pressed at once.

### Video

<video controls width="640" height="360">
  <source src="../UART.mp4" type="video/mp4">
</video>

## What I learned

- UART isn't just for printing to a terminal. It can also connect two microcontrollers.
- Hardware: crossed TX/RX and a common GND. Software: the same baud rate and format on both sides.
- A 1-character protocol was enough to send the state of 3 buttons.
- Sending only on state changes keeps the line from getting flooded.
- Limitation: there's no confirmation that a message arrived, so if a byte gets lost an LED could stay in the wrong state. It could be fixed by resending the state periodically or adding an ACK.