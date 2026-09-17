---
title: "GPIO from the SDK down to Hardware Registers (RP2350)"
date: 2026-08-31
tags: [embedded-systems, raspberry-pi-pico-2, gpio, registers, sio, c]
---

# GPIO from the SDK down to Hardware Registers (RP2350)

## Context

This activity used the RP2350's **SIO (Single-Cycle I/O)** block to see what `gpio_put()` actually does at the hardware level. Starting from a normal SDK blink, I rewrote it using registers directly (`sio_hw->gpio_set`, `sio_hw->gpio_clr`, `sio_hw->gpio_oe_set`, etc.), then used that to solve 4 exercises with 4 LEDs on GPIO2–GPIO5, driving the registers with bitmasks.

**Hardware connection:**

```
GPIO2 → LED0 → resistor → GND
GPIO3 → LED1 → resistor → GND
GPIO4 → LED2 → resistor → GND
GPIO5 → LED3 → resistor → GND
```

---

## Objective

Understand how the Raspberry Pi Pico SDK turns high-level functions (`gpio_put`, `gpio_set_dir`, etc.) into direct writes to the SIO registers, and use that to drive 4 LEDs with bitmasks in 4 different animations: binary counter, bouncing light, fill/drain, and outside-in fill.

## Materials and Components

- 1 × Raspberry Pi Pico 2 W
- 4 × LED (any color)
- 4 × 220 Ω resistor (current limiter)
- 1 × Protoboard
- Male-to-male jumper wires
- Micro-USB cable (programming and power)

## Circuit Diagram

The physical connection is the same for all 4 exercises; only the code changes. Each LED goes in series with its resistor between the GPIO and GND:

```
GPIO2 → LED0 → 220Ω resistor → GND
GPIO3 → LED1 → 220Ω resistor → GND
GPIO4 → LED2 → 220Ω resistor → GND
GPIO5 → LED3 → 220Ω resistor → GND
```

<!-- SPACE FOR CIRCUIT DIAGRAM IMAGE -->
![4-LED connection diagram](conexion_leds.svg)

The LED's long leg (anode) goes to the resistor/GPIO side, and the short leg (cathode) goes to GND.

---

## Exercise 1 — 4-bit Binary Counter

The 4 LEDs show a binary number counting from `0000` to `1111` (0 to 15), then repeat.

### Video

<!-- SPACE FOR VIDEO -->
<video controls width="640" height="360">
  <source src="ejercicio1.mp4" type="video/mp4">
</video>

### Code

```c
#include "pico/stdlib.h"
#include "hardware/structs/sio.h"

#define PINS 4// GP2, GP3, GP4, GP5

int main(void) {
    const uint32_t MASK = 0xF << 2;// bits 2,3,4,5 = 0b111100

    for (uint pin = 2; pin < 2 + PINS; pin++) {
        gpio_init(pin);
    }

    // configura mis 4 pines como salida de una sola vez
      sio_hw->gpio_oe_set = MASK;

    uint8_t sumador = 0;
    uint32_t numero_binario = 0;

    while(true) {

        numero_binario = (sumador << 2);// acomoda el numero en los pines

        sio_hw->gpio_clr = MASK; // apago
        sio_hw->gpio_set = numero_binario;// enciendo 

        sleep_ms(1000);

        sumador++;
        if(sumador > 15) {
            sumador = 0;
        }
    }
}
```

### How it works

`sumador` is a normal decimal counter, 0 to 15. Since GPIO2 is the first pin used and not bit 0 of the register, I shift the value left by 2 (`sumador << 2`) so its 4 bits land exactly on GPIO2–GPIO5, saved in `numero_binario`.

Each loop first clears all 4 pins (`gpio_clr = MASK`, so nothing from the last number stays on), then turns on only the current number's bits (`gpio_set = numero_binario`). `sumador++` moves the count forward, and once it goes past 15 it resets to 0, since that's the max 4 bits can hold.

---

## Exercise 2 — Bouncing Light

A single LED moves from one end of the row to the other and back, over and over.

### Video

<!-- SPACE FOR VIDEO -->
<video controls width="640" height="360">
  <source src="ejercicio2.mp4" type="ejercicio2/mp4">
</video>


### Code

```c
#include "pico/stdlib.h"
#include "hardware/structs/sio.h"

#define PINS 4      // GP2, GP3, GP4, GP5

int main(void) {
    const uint32_t MASK = 0xF << 2;  // bits 2,3,4,5 = 0b111100

    for (uint pin = 2; pin < 2 + PINS; pin++) {
        gpio_init(pin);
    }

    sio_hw->gpio_oe_set = MASK;

    uint8_t numero = 1;// inicia en 0001
    int subiendo = 1;//1=multiplica,0=divide

    while(true) {

        uint32_t numero_binario = ((uint32_t)numero << 2);

        sio_hw->gpio_clr = MASK;
        sio_hw->gpio_set = numero_binario;

        sleep_ms(500);

        if(subiendo){
            numero = numero * 2;      // 1-2-4-8
            if(numero == 8){
                subiendo = 0;         // llego al tope, ahora toca bajar
            }
        } else {
            numero = numero / 2;      // 8-4->2->1
            if(numero == 1){
                subiendo = 1;         // llego al piso, ahora toca subir
            }
        }
    }
}
```

### How it works

`numero` here isn't a counter, it's the actual value sent to the LEDs: 1 (`0001`), 2 (`0010`), 4 (`0100`), 8 (`1000`). Multiplying by 2 moves the single lit bit one step left, dividing by 2 moves it one step right — same shifting idea as before, just written with math instead of `<<`/`>>`.

The `subiendo` flag decides which one runs: while it's 1, `numero` keeps doubling (1→2→4→8); at 8 (GPIO5) it flips to 0 and starts halving instead (8→4→2→1); at 1 again (GPIO2) it flips back to 1. That flip at each end is what makes it bounce.

---

## Exercise 3 — Fill and Drain Animation

LEDs turn on one by one from one side until all 4 are lit, then turn off one by one until none are lit.

### Video

<!-- SPACE FOR VIDEO -->
<video controls width="640" height="360">
  <source src="ejercicio3.mp4" type="ejercicio3/mp4">
</video>


### Code

```c
#include "pico/stdlib.h"
#include "hardware/structs/sio.h"

#define PINS 4      // GP2, GP3, GP4, GP5

int main(void) {
    const uint32_t MASK = 0xF << 2;  // bits 2,3,4,5 = 0b111100

    for (uint pin = 2; pin < 2 + PINS; pin++) {
        gpio_init(pin);
    }

    sio_hw->gpio_oe_set = MASK;

    uint8_t numero = 0;//inicia en 0000
    int subiendo = 1;//1 =pone,0 =quita

    while(true) {

        uint32_t numero_binario = ((uint32_t)numero << 2);

        sio_hw->gpio_clr = MASK;
        sio_hw->gpio_set = numero_binario;

        sleep_ms(500);

        if(subiendo){
            numero = numero * 2 + 1;  
            if(numero == 15){
                subiendo = 0;      
            }
        } else {
            numero = numero / 2;       
            if(numero == 0){
                subiendo = 1;          
            }
        }
    }
}
```

### How it works

Here `numero` represents a **block of bits**, not just one lit LED. The key line is `numero * 2 + 1`: multiplying by 2 shifts every bit left, and `+1` turns on a new bit at the bottom, so each loop lights one more LED starting from GPIO2: 0 → 1 (`0001`) → 3 (`0011`) → 7 (`0111`) → 15 (`1111`).

Once `numero` hits 15 (all 4 on), `subiendo` flips to 0 and the code switches to `numero / 2`, which shifts bits right and drops the leftmost lit LED each time: 15 → 7 → 3 → 1 → 0. Hitting 0 flips `subiendo` back to 1 and it repeats — same bounce idea as exercise 2, just with a whole block of bits instead of one.

---

## Exercise 4 — Outside-in Fill

LEDs turn on starting from the outer edges and move toward the center, then turn off in the same outside-in order, on repeat.

### Video

<!-- SPACE FOR VIDEO -->
<video controls width="640" height="360">
  <source src="ejercicio4.mp4" type="ejercicio4/mp4">
</video>


### Code

```c
#include "pico/stdlib.h"
#include "hardware/structs/sio.h"

#define PINS 4  // GP2, GP3, GP4, GP5

int main(void) {
    const uint32_t MASK = 0xF << 2;   // bits 2,3,4,5 = 0b111100

    for (uint pin = 2; pin < 2 + PINS; pin++) {
        gpio_init(pin);
    }

    sio_hw->gpio_oe_set = MASK;

    const uint32_t MASK_EXTREMOS = ((1u << 0) | (1u << 3)) << 2; // GP2 y GP5

    const uint32_t MASK_CENTRO = ((1u << 1) | (1u << 2)) << 2; // GP3 Y GP4

   while(true) {

        for (int n = 0; n <= 3; n++) {

            uint32_t togl;

            if (n == 0 || n == 2) {
                togl = MASK_EXTREMOS;
            } else {
                togl = MASK_CENTRO;
            }

            sio_hw->gpio_togl = togl;

            sleep_ms(500);
        }

    }
}
```

### How it works

This pattern (`0000 → 1001 → 1111 → 0110 → 0000`) doesn't follow a simple math rule like the other exercises: the edges (GPIO2, GPIO5) light first, then the center (GPIO3, GPIO4) fills in, and turning off follows the same order. No `<<`, `+1`, or `*2` gets you those 4 steps directly.

So instead I built two fixed masks with `|`: `MASK_EXTREMOS` covers GPIO2 and GPIO5, `MASK_CENTRO` covers GPIO3 and GPIO4. Instead of forcing pins to a value, I use `sio_hw->gpio_togl`, which just **flips** whatever state the pins already had. That means the same 4-step `for` loop handles both turning on and turning off: LEDs start off and the toggle turns them on, and once all are lit the same code turns them off in the same order.

`if (n == 0 || n == 2)` picks which mask to use at each step: `MASK_EXTREMOS` on `n = 0` and `n = 2`, `MASK_CENTRO` on `n = 1` and `n = 3`, alternating over the 4 steps.

---

## Conclusion

Working with the SIO registers directly made it clear that SDK functions like `gpio_put()` or `gpio_set_dir()` are just a thin layer over simple bit operations: writes to `gpio_set`, `gpio_clr`, `gpio_togl`, and `gpio_oe_set`. Controlling all 4 LEDs with one instruction using masks, instead of handling each pin one at a time, showed why it's useful to operate several GPIOs at once.

Across the 4 exercises, I ended up using two different strategies to build a bit pattern:

1. **When there's a simple math rule** (exercises 1, 2, 3): the pattern can be calculated each loop with shifts or arithmetic (`<<`, `*2`, `/2`, `+1`), no need to hardcode each step.
2. **When the sequence is arbitrary** (exercise 4): there's no formula for it, so fixed masks plus `gpio_togl` is the cleanest option, since toggling flips the pin instead of forcing a value — letting one code block handle both turning on and off.

Overall, this activity showed how important it is to understand the hardware under the SDK: knowing which register bit maps to which physical pin is the base for building any GPIO pattern, instead of just relying on high-level functions.