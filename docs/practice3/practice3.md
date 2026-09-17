---
title: "LED Roulette with Interrupts (RP2350)"
date: 2026-09-13
tags: [embedded-systems, raspberry-pi-pico-2, gpio, interrupts, sio, c, roulette]
---

# LED Roulette with Interrupts (RP2350)

## Objective

Use GPIO interrupts (instead of polling) to build a 5-LED "roulette" that lights one LED at a time in sequence. One button checks, on press, whether the currently lit LED is the middle one — if so the game is "won" and all LEDs blink together until the same button is pressed again to restart. A second button cycles through three roulette speeds on every press.

## Materials and Components

- 1 × Raspberry Pi Pico 2 W
- 5 × LED
- 5 × 330 Ω resistor (current limiter for each LED)
- 2 × Push-button
- 1 × Protoboard
- Male-to-male jumper wires
- Micro-USB cable (programming and power)

## Circuit Diagram

```
GPIO2 → LED0 → 330Ω resistor → GND
GPIO3 → LED1 → 330Ω resistor → GND
GPIO4 → LED2 → 330Ω resistor → GND   (middle LED, the "winning" one)
GPIO5 → LED3 → 330Ω resistor → GND
GPIO6 → LED4 → 330Ω resistor → GND
GPIO7 → Button 1 (win / restart) → GND   (internal pull-up)
GPIO8 → Button 2 (speed)         → GND   (internal pull-up)
```

![connection diagram](../diagrama1.png)
/// caption
Diagrama de conexión para la ruleta de LEDs
///

Unlike the logic-gate exercises, both buttons here use the microcontroller's internal **pull-up** (`gpio_pull_up()`), so each pin reads `1` when not pressed and drops to `0` when pressed. The interrupt is set on `GPIO_IRQ_EDGE_RISE`, so it fires the moment the button is released back up.

## Source Code

```c
#include "pico/stdlib.h"
#include "hardware/structs/sio.h"
#include "hardware/gpio.h"
#include <stdio.h>


#define BOTON_PIN 7      
#define BOTON_PIN2 8     


const uint32_t MASK_LED = 0x1F << 2; 


volatile bool gane = false;   
volatile int counter = 0;     
volatile int vel = 500;       


void stop_callback(uint gpio, uint32_t events)
{
    
    if (gpio == BOTON_PIN && (events & GPIO_IRQ_EDGE_RISE))
    {
        printf("Botón de ganar presionado\n");

        if (!gane) 
        {
            
            if (counter == 2)
            {
                printf("¡Ganaste! Empieza el parpadeo\n");
                gane = true; 
            }
            else
            {
               
                printf("Fallaste, sigue jugando\n");
            }
        }
        else 
        {
            printf("Reiniciando el juego\n");
            gane = false;   
            counter = 0;    
        }
    }

   
    gpio_acknowledge_irq(gpio, events);
}


void boton_velocidad(uint gpio, uint32_t events)
{
    if (gpio == BOTON_PIN2 && (events & GPIO_IRQ_EDGE_RISE))
    {
        printf("Botón de velocidad presionado\n");

    
        if (vel == 500)
        {
            vel = 250; 
        }
        else if (vel == 250)
        {
            vel = 100; 
        }
        else if (vel == 100)
        {
            vel = 500; 
        }
    }

    gpio_acknowledge_irq(gpio, events); 
}

void escoge_boton(uint gpio, uint32_t events)
{
    if (gpio == BOTON_PIN)
    {
        stop_callback(gpio, events);
    }
    else if (gpio == BOTON_PIN2)
    {
        boton_velocidad(gpio, events);
    }
}

int main(void)
{
    stdio_init_all(); 

    
    gpio_init(2);
    gpio_init(3);
    gpio_init(4);
    gpio_init(5);
    gpio_init(6);
    gpio_init(BOTON_PIN);
    gpio_init(BOTON_PIN2);

 
    sio_hw->gpio_oe_set = MASK_LED;

   
    sio_hw->gpio_oe_clr = (1u << BOTON_PIN) | (1u << BOTON_PIN2);

    // Pull-up: el pin normalmente está en "1" (alto), y baja a "0"
    // cuando presionas el botón (conecta a GND)
    gpio_pull_up(BOTON_PIN);
    gpio_pull_up(BOTON_PIN2);

    
    gpio_set_irq_enabled_with_callback(BOTON_PIN, GPIO_IRQ_EDGE_RISE, true, &escoge_boton);

    
    gpio_set_irq_enabled(BOTON_PIN2, GPIO_IRQ_EDGE_RISE, true);

    
    while (true)
    {
        if (gane)
        {
            
            sio_hw->gpio_set = MASK_LED; 
            sleep_ms(200);
            sio_hw->gpio_clr = MASK_LED; 
            sleep_ms(200);
            
        }
        else
        {
           
            sio_hw->gpio_clr = MASK_LED;             
            sio_hw->gpio_set = (1u << (2 + counter));  
            sleep_ms(vel);                             

            printf("GPIO: %d \n", sio_hw->gpio_in);

            counter++;          
            if (counter > 4)   
                counter = 0;    
        }
    }
}
```

## How it works

The roulette itself is plain polling in the main loop: `counter` goes from 0 to 4, redrawn every pass with `1u << (2 + counter)` to light exactly one LED on GPIO2–GPIO6, and the delay between steps comes from the shared `vel` variable. What's different from the earlier labs is that the buttons aren't read by polling `gpio_in` anymore — they're handled with hardware interrupts.

The Pico SDK only lets you register one global callback for GPIO interrupts (`gpio_set_irq_enabled_with_callback`), so `escoge_boton` works as a router: it checks which `gpio` fired and forwards it to `stop_callback` or `boton_velocidad`. The second button is added afterward with a plain `gpio_set_irq_enabled`, since the callback is already set.

`stop_callback` only sets `gane = true` if `counter == 2` at the moment the button is pressed — that's the middle LED (GPIO4). If a different LED was on, nothing happens and the roulette keeps going. If the game was already won, pressing the same button resets `gane` to `false` and `counter` to `0`, restarting the roulette. `boton_velocidad` just cycles `vel` through 500 → 250 → 100 ms on every press.

All three shared variables (`gane`, `counter`, `vel`) are marked `volatile`, since they get written inside an interrupt and read inside `main()`'s loop — without `volatile` the compiler could cache their value and miss the change made by the interrupt. Every callback also calls `gpio_acknowledge_irq()` at the end, which clears the interrupt flag in hardware; skipping it leaves the pin stuck re-firing the same interrupt.

## Results

The LED sequence runs as a continuous roulette. Pressing the win button while the middle LED (GPIO4) is lit switches all 5 LEDs into a synchronized blink; pressing it again while blinking resets the game and the roulette starts over from the first LED. Pressing the win button while any other LED is lit does nothing. The speed button reliably cycles the roulette through slow, medium, and fast on each press, whether the game is currently won or running.

### Video

<video controls width="640" height="360">
  <source src="../ruleta.mp4" type="video/mp4">
</video>

## Conclusions

Moving from polling (like the moving-LED lab, which used flags `f1`/`f2` to catch a single press inside the main loop) to interrupts meant the main loop doesn't have to check button state at all anymore — it just reacts to `gane` and `vel` whenever they change. This also made something clear: since only one callback can be registered per core, I needed a router function to dispatch by pin, and any variable shared between an interrupt and `main()` has to be `volatile` or the change might never show up outside the interrupt. One thing this activity didn't solve yet: without software debounce, a single physical press can sometimes register as more than one edge, which would need a time-based check (like comparing against `time_us_32()`) to fix properly.