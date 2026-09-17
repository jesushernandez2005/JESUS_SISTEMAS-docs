---
title: "Buttons and Logic Gates with GPIO (RP2350)"
date: 2026-09-06
tags: [embedded-systems, raspberry-pi-pico-2, gpio, registers, sio, c, logic-gates]
---

# Buttons and Logic Gates with GPIO (RP2350)

## Objective

Use two push-buttons as digital inputs to build AND, OR, and XOR logic in hardware by reading `sio_hw->gpio_in` and checking bits, and use the same buttons to move a lit LED across a row of 4 LEDs.

## Materials and Components

- 1 × Raspberry Pi Pico 2 W
- 4 × LED (for the moving-LED part)
- 5 × 330 Ω resistor (one per LED)
- 2 × Push-button
- 1 × Protoboard
- Male-to-male jumper wires
- Micro-USB cable (programming and power)

## Circuit Diagram

**AND / OR / XOR** (one output LED, two input buttons):

```
GPIO2 → LED → 220Ω resistor → GND
GPIO6 → Button A → GND     (internal pull-down)
GPIO7 → Button B → GND     (internal pull-down)
```

![connection diagram](../diagrama.png)
/// caption
Diagrama de conexión para las compuertas AND, OR y XOR
///

```
GPIO2 → LED0 → 220Ω resistor → GND
GPIO3 → LED1 → 220Ω resistor → GND
GPIO4 → LED2 → 220Ω resistor → GND
GPIO5 → LED3 → 220Ω resistor → GND
GPIO6 → Left button  → GND   (internal pull-down)
GPIO7 → Right button → GND   (internal pull-down)
```

<!-- SPACE FOR CIRCUIT DIAGRAM IMAGE (MOVING LED) -->

Both buttons use the Pico's internal pull-down (`gpio_pull_down()`), so a pin reads `0` when the button is not pressed and `1` when it is. No external resistor is needed on the buttons.

---

## AND Gate

The LED turns on only when **both** buttons are pressed at the same time.

### Source Code

```c
#include "pico/stdlib.h"
#include "hardware/structs/sio.h"
#include <stdio.h>

int main(void) {
    stdio_init_all();

    const uint32_t MASK_LED = 1u<< 2;
    const uint32_t MASK_BOTONES = (1u << 6) | (1u << 7);

    gpio_init(2);
    gpio_init(6);
    gpio_init(7);

    sio_hw->gpio_oe_set = MASK_LED;
    sio_hw->gpio_oe_clr = MASK_BOTONES;

    gpio_pull_down(6);
    gpio_pull_down(7);

    while (true) {

        uint32_t entaadas = sio_hw->gpio_in;

    
        int resultado = (entaadas & MASK_BOTONES);
        printf("Botones: %d\n", resultado);
        if (entaadas & 1<<6 && entaadas & 1<<7) {
            sio_hw->gpio_set = MASK_LED;
        } else {
            sio_hw->gpio_clr = MASK_LED;
        }
    }
}
```

### How it works

`gpio_in` reads all 32 pins at once as one number. I mask out the two button bits (`resultado`) just to print them over serial and check what's coming in. The actual decision happens in the `if`, where I check each button bit on its own: `entaadas & 1<<6` and `entaadas & 1<<7` both have to be non-zero for the LED to turn on — that only happens when both buttons are down, which is AND logic.

| A | B | LED (A AND B) |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | 0 |
| 1 | 0 | 0 |
| 1 | 1 | **1** |

### Results

The LED only lights up while both buttons are held down together; letting go of either one turns it off right away. The serial monitor also prints the masked button value every loop, which I used to confirm the register was reading what I expected.

### Conclusions

Checking each button bit separately in the `if` worked fine and honestly reads closer to normal AND logic than comparing against a combined mask. Adding the `printf` helped me trust the LED's behavior because I could see the raw bits while testing.

### Video

<video controls width="640" height="360">
  <source src="../And2.mp4" type="video/mp4">
</video>

---

## OR Gate

The LED turns on if **at least one** button is pressed.

### Source Code

```c
#include "pico/stdlib.h"
#include "hardware/structs/sio.h"
#include <stdio.h>

int main(void) {
    stdio_init_all();

    const uint32_t MASK_LED = 1u<< 2;
    const uint32_t MASK_BOTONES = (1u << 6) | (1u << 7);

    gpio_init(2);
    gpio_init(6);
    gpio_init(7);

    sio_hw->gpio_oe_set = MASK_LED;
    sio_hw->gpio_oe_clr = MASK_BOTONES;

    gpio_pull_down(6);
    gpio_pull_down(7);

    while (true) {

        uint32_t entaadas = sio_hw->gpio_in;

    
        int resultado = (entaadas & MASK_BOTONES);
        printf("Botones: %d\n", resultado);
        if (entaadas & MASK_BOTONES) {
            sio_hw->gpio_set = MASK_LED;
        } else {
            sio_hw->gpio_clr = MASK_LED;
        }
    }
}
```

### How it works

Only the `if` condition changes from the AND version. Instead of requiring both bits separately, it just needs `entaadas & MASK_BOTONES` to be non-zero — that's true as soon as either button bit is set, which is exactly what OR means. I kept the same `printf` from the AND code to check the raw value while testing.

| A | B | LED (A OR B) |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | **1** |
| 1 | 0 | **1** |
| 1 | 1 | **1** |

### Results

Pressing either button, or both, lights the LED. It only stays off when neither is pressed.

### Conclusions

Going from AND to OR just meant switching the condition to check the combined mask instead of both bits one by one — same register read, different comparison.

### Video

<video controls width="640" height="360">
  <source src="../OR.mp4" type="video/mp4">
</video>

---

## XOR Gate

The LED turns on only when **exactly one** of the two buttons is pressed.

### Source Code

```c
#include "pico/stdlib.h"
#include "hardware/structs/sio.h"
#include <stdio.h>

int main(void) {
    stdio_init_all();

    const uint32_t MASK_LED = 1u<< 2;
    const uint32_t MASK_BOTONES = (1u << 6) | (1u << 7);

    gpio_init(2);
    gpio_init(6);
    gpio_init(7);

    sio_hw->gpio_oe_set = MASK_LED;
    sio_hw->gpio_oe_clr = MASK_BOTONES;

    gpio_pull_down(6);
    gpio_pull_down(7);

    while (true) {

        uint32_t entaadas = sio_hw->gpio_in;
        int a=(entaadas & (1u << 6))!=0;
        int b=(entaadas & (1u << 7))!=0;

        int resultado = (a ^ b);
        printf("Botones: %d\n", resultado);
        if (resultado) {
            sio_hw->gpio_set = MASK_LED;
        } else {
            sio_hw->gpio_clr = MASK_LED;
        }
    }
}
```

### How it works

XOR has two valid cases (`01` and `10`), so a single mask check isn't enough here. I pull each button out on its own as a clean 0 or 1 (`a` and `b`), then combine them with `^`, which gives 1 only when they're different. This time `resultado` is the real XOR output, not just for debugging — it drives both the `printf` and the LED.

| A | B | LED (A XOR B) |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | **1** |
| 1 | 0 | **1** |
| 1 | 1 | 0 |

### Results

The LED turns on for A-only or B-only, and stays off when both or neither button is pressed.

### Conclusions

XOR needed a different approach than AND/OR since no single mask covers both valid combinations. Splitting the bits into `a` and `b` and using `^` was the simplest way to match its truth table.

### Video

<video controls width="640" height="360">
  <source src="../XOR.mp4" type="video/mp4">
</video>

---

## Moving LED

Two buttons shift a single lit LED across 4 LEDs (GPIO2–GPIO5), wrapping around at the ends instead of stopping.

### Source Code

```c
#include "pico/stdlib.h"
#include "hardware/structs/sio.h"
#include <stdio.h>

int main(void) {
    stdio_init_all();

    const uint32_t MASK_LED = 0xF << 2;// bits 2,3,4,5 = 0b111100
    const uint32_t MASK_BOTONES = (1u << 6) | (1u << 7);

    gpio_init(2);
    gpio_init(3);
    gpio_init(4);
    gpio_init(5);
    gpio_init(6);
    gpio_init(7);

    sio_hw->gpio_oe_set = MASK_LED;
    sio_hw->gpio_oe_clr = MASK_BOTONES;

    gpio_pull_down(6);
    gpio_pull_down(7);

    int counter = 0;
    int f1= 0;
    int f2= 0;
while (true) {
        uint32_t entradas = sio_hw->gpio_in;
        int a = (entradas & (1u << 6)) != 0;
        int b = (entradas & (1u << 7)) != 0;

       
        if (a && !f1) {
            counter++;
            if (counter > 3) counter = 0;   
            f1 = 1;
        } else if (!a && f1) {
            f1 = 0;
        }

        
        if (b && !f2) {
            counter--;
            if (counter < 0) counter = 3;   
            f2 = 1;
        } else if (!b && f2) {
            f2 = 0;
        }

        
        sio_hw->gpio_clr = MASK_LED;             
        sio_hw->gpio_set = (1u << (2 + counter));  

        sleep_ms(15); 
    }
}
```

### How it works

`counter` (0–3) tracks which LED is lit, and I redraw it every loop with `1u << (2 + counter)` so it lands on GPIO2–GPIO5. Instead of blocking with a `while` until the button is released, each button gets its own flag (`f1` for GPIO6, `f2` for GPIO7) so it only moves once per press: the flag goes high as soon as the button is seen pressed, and only resets once the button is read as released again. `sleep_ms(15)` just slows down the polling a bit. Button 6 increases `counter` and wraps from 3 back to 0; button 7 decreases it and wraps from 0 back to 3.

### Results

The LED moves one step per press in either direction, and instead of stopping at GPIO2 or GPIO5, it wraps around to the other end when a button is held past the limit.

### Conclusions

Switching from a blocking `while (gpio_get(...))` to a press/release flag per button gets the same one-step-per-press behavior without freezing the main loop, and it made adding the wrap-around a lot easier than trying to clamp at the edges.

### Video

<video controls width="640" height="360">
  <source src="../TOY.mp4" type="video/mp4">
</video>