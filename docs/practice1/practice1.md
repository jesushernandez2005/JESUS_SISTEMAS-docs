---
title: "GPIO from the SDK down to Hardware Registers (RP2350)"
date: 2026-08-31
tags: [embedded-systems, raspberry-pi-pico-2, gpio, registers, sio, c]
---

# GPIO from the SDK down to Hardware Registers (RP2350)

## Context

This activity worked with the RP2350's **SIO (Single-Cycle I/O)** block to understand what `gpio_put()` actually does at the hardware level. Starting from an SDK-based blink, it was rewritten using registers (`sio_hw->gpio_set`, `sio_hw->gpio_clr`, `sio_hw->gpio_oe_set`, etc.), then applied to solve 4 exercises with 4 LEDs on GPIO2–GPIO5, driving the registers directly with bitmasks.

**Hardware connection:**

```
GPIO2 → LED0 → resistor → GND
GPIO3 → LED1 → resistor → GND
GPIO4 → LED2 → resistor → GND
GPIO5 → LED3 → resistor → GND
```

---

## Objective

Understand how the Raspberry Pi Pico SDK translates high-level functions (`gpio_put`, `gpio_set_dir`, etc.) into direct writes to the SIO peripheral's registers, and use that to drive 4 LEDs with bitmasks, producing different animation patterns (binary counter, bouncing light, progressive fill/drain, and outside-in fill).

## Materials and Components

- 1 × Raspberry Pi Pico 2 W
- 4 × LED (any color)
- 4 × 220 Ω resistor (current limiter)
- 1 × Protoboard
- Male-to-male jumper wires
- Micro-USB cable (programming and power)

## Circuit Diagram

The physical connection is the same for all 4 exercises; only the code controlling the pins changes. Each LED is wired in series with its resistor between the corresponding GPIO and GND:

```
GPIO2 → LED0 → 220Ω resistor → GND
GPIO3 → LED1 → 220Ω resistor → GND
GPIO4 → LED2 → 220Ω resistor → GND
GPIO5 → LED3 → 220Ω resistor → GND
```

<!-- SPACE FOR CIRCUIT DIAGRAM IMAGE -->
![4-LED connection diagram](conexion_leds.svg)

The LED's long leg (anode) goes to the resistor/GPIO side, and the short leg (cathode) goes to the GND rail.

---

## Exercise 1 — 4-bit Binary Counter

The 4 LEDs represent a binary number counting from `0000` to `1111` (0 to 15), repeating.

### Video

<!-- SPACE FOR VIDEO -->
<iframe width="640" height="360" 
  src="https://youtube.com/shorts/K7jKG_uqHXw?si=jHdKx_hQ5cUBY4t9" 
  title="EJERCICIO_1" 
  frameborder="0" 
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" 
  allowfullscreen>
</iframe>

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

### Logic Used

`sumador` is the decimal counter (0 to 15). Since GPIO2 is the first physical pin used, not bit 0 of the register, its value has to be shifted left 2 places (`sumador << 2`) so its 4 bits land exactly on GPIO2–GPIO5, stored in `numero_binario`.

Each loop first clears the whole group (`gpio_clr = MASK`, avoiding leftover bits from the previous number), then sets only the current number's bits (`gpio_set = numero_binario`). `sumador++` advances the count, and `if (sumador > 15) sumador = 0` wraps it back to zero once it exceeds what 4 bits can hold.

---

## Exercise 2 — Bouncing Light

A single lit LED that moves from one end of the array to the other and back, continuously.

### Video

<!-- SPACE FOR VIDEO -->
<iframe width="640" height="360" 
  src="https://youtube.com/shorts/SKljldsNEv0?si=P1q_SWlPBcciRE_c" 
  title="EJERCICIO_2" 
  frameborder="0" 
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" 
  allowfullscreen>
</iframe>

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

### Logic Used

Here `numero` isn't a position or index — it's the actual value written to the LEDs: 1 (`0001`), 2 (`0010`), 4 (`0100`), 8 (`1000`). Multiplying by 2 shifts the single lit bit one step left; dividing by 2 shifts it one step right — same bit-shifting idea as before, just expressed arithmetically instead of with `<<`/`>>`.

The `subiendo` flag decides which operation runs: while it's 1, `numero` doubles (1→2→4→8); on hitting 8 (GPIO5), it flips to 0 and `numero` starts halving (8→4→2→1); on hitting 1 again (GPIO2), it flips back. That flag flip at each end is what creates the "bounce."

---

## Exercise 3 — Fill and Drain Animation

LEDs light up progressively from one side until all are on, then turn off progressively until all are off.

### Video

<!-- SPACE FOR VIDEO -->
<iframe width="640" height="360" 
  src="https://youtube.com/shorts/_psb46dYPCc?si=sW8lfIzUncVOYMYw" 
  title="EJERCICIO_3" 
  frameborder="0" 
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" 
  allowfullscreen>
</iframe>

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

### Logic Used

Here `numero` represents a **block of consecutive bits** rather than a single lit bit. The trick is `numero * 2 + 1`: multiplying by 2 shifts every bit left, and the `+1` turns on a new bit in the lowest position, so each loop "lights one more LED" starting from GPIO2: 0 → 1 (`0001`) → 3 (`0011`) → 7 (`0111`) → 15 (`1111`).

Once `numero` hits 15 (all 4 LEDs on), `subiendo` flips to 0 and the opposite operation kicks in: `numero / 2`, which shifts bits right, always dropping the leftmost lit LED: 15 → 7 → 3 → 1 → 0. Hitting 0 flips `subiendo` back to 1 and the cycle repeats — same bounce idea as exercise 2, but moving a whole block of bits instead of just one.

---

## Exercise 4 — Outside-in Fill

LEDs turn on starting from the outer edges toward the center, then turn off following the same outside-in order, continuously.

### Video

<!-- SPACE FOR VIDEO -->
<iframe width="640" height="360" 
  src="https://youtube.com/shorts/gKS8zamVWCE?si=05tgUziXsB-EglEn" 
  title="EJERCICIO_4" 
  frameborder="0" 
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" 
  allowfullscreen>
</iframe>

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

### Logic Used

This pattern (`0000 → 1001 → 1111 → 0110 → 0000`) doesn't follow a simple arithmetic rule like exercises 1–3: the edges (GPIO2, GPIO5) light first, then the center fills in (GPIO3, GPIO4), and the same order applies when turning off. No single `<<`, `+1`, or `*2` produces those 4 steps directly.

So instead, two fixed masks are built with `|`: `MASK_EXTREMOS` combines GPIO2 and GPIO5, `MASK_CENTRO` combines GPIO3 and GPIO4. Rather than forcing pins to a fixed value, `sio_hw->gpio_togl` is used, which **flips** whatever state the pins already had. That lets the same 4-step `for` loop handle both turning on and turning off: LEDs start off and the toggle turns them on; once all are lit, the same code block turns them off in the same order.

`if (n == 0 || n == 2)` decides which mask applies at each step: `MASK_EXTREMOS` on `n = 0` and `n = 2`, `MASK_CENTRO` on `n = 1` and `n = 3`, alternating across the 4 iterations.

---

## Conclusion

Working directly with the SIO registers made it clear that SDK functions like `gpio_put()` or `gpio_set_dir()` are just a thin abstraction over simple bit operations: writes to `gpio_set`, `gpio_clr`, `gpio_togl`, and `gpio_oe_set`. Controlling 4 LEDs with a single instruction (via masks) instead of handling each pin separately showed the advantage of operating several GPIOs at once, efficiently.

Across the 4 exercises, two distinct strategies emerged for generating a bit pattern:

1. **When a constant mathematical rule exists** (exercises 1, 2, 3): the pattern can be computed each iteration with arithmetic or shift operations (`<<`, `*2`, `/2`, `+1`), avoiding a hand-written pattern.
2. **When the sequence is arbitrary** (exercise 4): no simple formula exists, so the clearest solution is fixed masks combined with `gpio_togl`, taking advantage of the fact that this register flips the pin's state instead of forcing it — letting the same code block serve for both turning on and off.

Overall, the activity reinforced how important it is to understand the hardware beneath the SDK: knowing which register bit maps to which physical pin is the foundation for designing any GPIO control pattern, without relying solely on high-level functions.