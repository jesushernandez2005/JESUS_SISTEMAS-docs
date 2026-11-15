---
title: "Two-Way UART Communication Between Two Picos (RP2350)"
date: 2026-10-28
tags: [embedded-systems, raspberry-pi-pico-2, uart, serial-communication, gpio, c]
---

# Two-Way UART Communication Between Two Picos (RP2350)

## Objective

Connect two Raspberry Pi Pico 2 W boards through UART so they control each other. Each board has 3 buttons and 3 LEDs. Pressing a button on one board turns on the matching LED on the other board, and releasing it turns that LED off. Communication works in both directions at the same time, and both boards run exactly the same program.

## Materials and Components

- 2 × Raspberry Pi Pico 2 W
- 6 × LED (3 per board)
- 6 × 330 Ω resistor (current limiter for each LED)
- 6 × Push-button (3 per board)
- 2 × Protoboard
- Male-to-male jumper wires
- 2 × Micro-USB cable (programming and power)

## Circuit Diagram

Each board is wired the same way:

```
GPIO2 → LED A → 330Ω resistor → GND
GPIO3 → LED B → 330Ω resistor → GND
GPIO4 → LED C → 330Ω resistor → GND
GPIO5 → Button A → GND   (internal pull-up)
GPIO6 → Button B → GND   (internal pull-up)
GPIO7 → Button C → GND   (internal pull-up)
```

The two boards are connected to each other with crossed UART lines and a common ground:

```
Pico 1 GPIO0 (TX) → Pico 2 GPIO1 (RX)
Pico 1 GPIO1 (RX) ← Pico 2 GPIO0 (TX)
Pico 1 GND        ↔ Pico 2 GND
```


The TX and RX lines have to be **crossed**: what one board transmits has to arrive at the receiving pin of the other. The shared GND is also essential, because UART levels are measured relative to ground; without a common reference the receiver can't tell a `1` from a `0`.

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

Both boards configure UART0 on GPIO0 (TX) and GPIO1 (RX) at 115200 baud with the 8N1 format (8 data bits, no parity, 1 stop bit). Both sides must use exactly the same settings, since UART is asynchronous: there is no shared clock line, so each receiver relies on the agreed baud rate to know when to sample every bit.

The boards talk using a simple one-character protocol. An uppercase letter means "button pressed / LED on" and a lowercase letter means "button released / LED off":

| Character | Sent when… | Effect on the other board |
|-----------|------------|---------------------------|
| `A` / `a` | Button A pressed / released | LED A on / off |
| `B` / `b` | Button B pressed / released | LED B on / off |
| `C` / `c` | Button C pressed / released | LED C on / off |

**Transmission.** Each button has its own flag (`flag_a`, `flag_b`, `flag_c`) that remembers whether it was already pressed. A character is sent only when the state *changes*: the uppercase letter when the pin drops to `0` while the flag is `0`, and the lowercase letter when the pin returns to `1` while the flag is `1`. Without these flags the board would send `'A'` every 10 ms for as long as the button is held, flooding the line with repeated messages. `uart_putc()` sends a single byte, which is all this protocol needs.

**Reception.** Every pass of the loop, `uart_is_readable()` checks whether bytes have arrived in the UART's receive FIFO. The inner `while` reads them one by one with `uart_getc()` until the FIFO is empty, so no messages are lost if several arrived between loop passes (for example, two buttons pressed at almost the same time). Each character turns on or off its corresponding LED with `gpio_put()`, and any other character is simply ignored.

Since sending and receiving happen in the same loop and UART has separate TX and RX lines, both boards can transmit and receive at the same time (full-duplex). That's why the same program works on both boards without needing a "master" and a "slave". The `sleep_ms(10)` at the end limits the loop to about 100 passes per second, which is fast enough to feel instant and also helps a bit against button bounce.

## Results

Pressing a button on one board lights the matching LED on the other board, and the LED stays on exactly as long as the button is held; releasing it turns the LED off. The same thing works in the opposite direction at the same time, and several buttons can be held at once on either board, each controlling its own LED independently.

### Video

<video controls width="640" height="360">
  <source src="UART.mp4" type="video/mp4">
</video>

## Conclusions

This activity moved UART from a one-way debugging output (like sending sensor readings to a terminal) to actual communication between two microcontrollers. The key hardware lessons were crossing TX and RX and sharing GND, and on the software side, that both boards must agree on the same baud rate and frame format since UART has no clock line. Defining a tiny protocol, uppercase for "on" and lowercase for "off", was enough to transfer the state of three buttons with single bytes. Sending only on state changes, using the same flag technique as earlier polling labs, kept the line from being flooded and made each message meaningful. One limitation is that there is no confirmation that a message arrived: if a byte were lost or corrupted, an LED could stay in the wrong state until the next press. A more robust version could periodically resend the current state of all buttons or add a simple acknowledgment.