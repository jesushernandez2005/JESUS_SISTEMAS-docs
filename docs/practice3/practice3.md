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

<!-- SPACE FOR CIRCUIT DIAGRAM IMAGE -->

Unlike the previous logic-gate exercises, both buttons here rely on the microcontroller's internal **pull-up** (`gpio_pull_up()`), so each pin reads `1` when unpressed and drops to `0` when pressed — the interrupt is configured on `GPIO_IRQ_EDGE_RISE`, so it fires the instant the button is released back up.

## Source Code

```c
#include "pico/stdlib.h"
#include "hardware/structs/sio.h"
#include "hardware/gpio.h"
#include <stdio.h>

// Pines donde están conectados los botones
#define BOTON_PIN 7      // Botón para "ganar" / reiniciar
#define BOTON_PIN2 8     // Botón para cambiar la velocidad

// Máscara para prender/apagar los 5 LEDs de un solo golpe
// Los LEDs están en GPIO 2,3,4,5,6 -> bits 2 al 6 encendidos
const uint32_t MASK_LED = 0x1F << 2; // 0b01111100

// Variables "compartidas" entre el loop principal y las interrupciones.
// volatile = "oye compilador, esto puede cambiar en cualquier momento
// por fuera del código normal, no la guardes en caché, léela siempre de la RAM"
volatile bool gane = false;   // true = ya ganaste, LEDs parpadeando
volatile int counter = 0;     // qué LED está prendido ahorita (0 a 4)
volatile int vel = 500;       // qué tan rápido se prenden los LEDs (ms)

// ---------------------------------------------------------
// Esta función se ejecuta SOLO cuando se presiona BOTON_PIN
// ---------------------------------------------------------
void stop_callback(uint gpio, uint32_t events)
{
    // Doble chequeo: que sí sea el botón correcto Y que sí sea
    // el evento correcto (flanco de subida, o sea, se soltó)
    if (gpio == BOTON_PIN && (events & GPIO_IRQ_EDGE_RISE))
    {
        printf("Botón de ganar presionado\n");

        if (!gane) // si todavía NO habíamos ganado...
        {
            // counter == 2 significa que el LED de en medio (GPIO 4)
            // es el que está prendido en este instante
            if (counter == 2)
            {
                printf("¡Ganaste! Empieza el parpadeo\n");
                gane = true; // avisamos al loop principal que ya ganamos
            }
            else
            {
                // No era el LED de en medio, no pasa nada,
                // el juego sigue como si nada
                printf("Fallaste, sigue jugando\n");
            }
        }
        else // si ya habíamos ganado antes...
        {
            printf("Reiniciando el juego\n");
            gane = false;   // regresamos al modo normal (ruleta)
            counter = 0;    // reiniciamos desde el primer LED
        }
    }

    // OBLIGATORIO: le decimos al hardware "ya atendí esta interrupción,
    // bájale la bandera". Si no lo haces, se vuelve a disparar sola.
    gpio_acknowledge_irq(gpio, events);
}

// ---------------------------------------------------------
// Esta función se ejecuta SOLO cuando se presiona BOTON_PIN2
// ---------------------------------------------------------
void boton_velocidad(uint gpio, uint32_t events)
{
    if (gpio == BOTON_PIN2 && (events & GPIO_IRQ_EDGE_RISE))
    {
        printf("Botón de velocidad presionado\n");

        // Vamos rotando entre 3 velocidades: 500 -> 250 -> 100 -> 500...
        if (vel == 500)
        {
            vel = 250; // más rápido
        }
        else if (vel == 250)
        {
            vel = 100; // más rápido todavía
        }
        else if (vel == 100)
        {
            vel = 500; // regresamos a lento
        }
    }

    gpio_acknowledge_irq(gpio, events); // mismo trámite de siempre
}

// ---------------------------------------------------------
// El Pico SDK solo permite UN callback global de interrupciones
// de GPIO. Por eso esta función actúa como "recepcionista":
// revisa QUÉ pin mandó la interrupción y manda la llamada
// a la función correcta.
// ---------------------------------------------------------
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
    stdio_init_all(); // para que funcione el printf por USB/serial

    // Inicializamos cada pin que vamos a usar
    gpio_init(2);
    gpio_init(3);
    gpio_init(4);
    gpio_init(5);
    gpio_init(6);
    gpio_init(BOTON_PIN);
    gpio_init(BOTON_PIN2);

    // Los 5 LEDs son SALIDAS (el Pico manda voltaje hacia ellos)
    sio_hw->gpio_oe_set = MASK_LED;

    // Los botones son ENTRADAS (el Pico solo lee su estado)
    sio_hw->gpio_oe_clr = (1u << BOTON_PIN) | (1u << BOTON_PIN2);

    // Pull-up: el pin normalmente está en "1" (alto), y baja a "0"
    // cuando presionas el botón (conecta a GND)
    gpio_pull_up(BOTON_PIN);
    gpio_pull_up(BOTON_PIN2);

    // Aquí registramos la ÚNICA función que atiende TODAS las
    // interrupciones de GPIO, y de paso activamos la interrupción
    // del primer botón
    gpio_set_irq_enabled_with_callback(BOTON_PIN, GPIO_IRQ_EDGE_RISE, true, &escoge_boton);

    // El segundo botón solo necesita "prenderse", ya que el
    // callback global ya quedó registrado arriba
    gpio_set_irq_enabled(BOTON_PIN2, GPIO_IRQ_EDGE_RISE, true);

    // ---------------------------------------------------------
    // Loop principal: aquí vive la "ruleta" de LEDs
    // ---------------------------------------------------------
    while (true)
    {
        if (gane)
        {
            // Modo "ganaste": todos los LEDs parpadean juntos
            sio_hw->gpio_set = MASK_LED; // prende los 5 LEDs
            sleep_ms(200);
            sio_hw->gpio_clr = MASK_LED; // apaga los 5 LEDs
            sleep_ms(200);
            // Este ciclo se repite solito hasta que el botón
            // vuelva a poner gane = false
        }
        else
        {
            // Modo normal: la ruleta va prendiendo un LED a la vez
            sio_hw->gpio_clr = MASK_LED;              // apaga todos primero
            sio_hw->gpio_set = (1u << (2 + counter));  // prende solo el LED actual
            sleep_ms(vel);                             // espera según la velocidad

            printf("GPIO: %d \n", sio_hw->gpio_in);

            counter++;          // pasamos al siguiente LED
            if (counter > 4)    // si ya llegamos al último (GPIO 6)...
                counter = 0;    // ...regresamos al primero (GPIO 2)
        }
    }
}
```

## System Operation

The roulette itself is simple polling in the main loop: `counter` walks from 0 to 4, redrawn each pass with `1u << (2 + counter)` to light exactly one of GPIO2–GPIO6, and the delay between steps is set by the shared `vel` variable. What makes this activity different from the earlier logic-gate and moving-LED labs is that the buttons are no longer read by polling `gpio_in` inside the loop — they're handled by hardware interrupts.

Only one callback can be registered globally for GPIO interrupts (`gpio_set_irq_enabled_with_callback`), so `escoge_boton` acts as a router: it checks which `gpio` triggered the interrupt and forwards it to `stop_callback` or `boton_velocidad`. The second button is added afterward with a plain `gpio_set_irq_enabled`, since the callback is already registered.

`stop_callback` only sets `gane = true` if `counter == 2` at the moment the button is pressed — that's the middle LED (GPIO4). If a different LED was lit, nothing happens and the roulette keeps running. If the game was already won, the same button press resets `gane` to `false` and `counter` back to `0`, restarting the roulette. `boton_velocidad` just rotates `vel` through 500 → 250 → 100 ms on every press.

All three shared variables (`gane`, `counter`, `vel`) are declared `volatile`, since they're written inside an ISR and read inside `main()`'s loop — without it the compiler could cache their value and never notice the change made by an interrupt. Every callback also calls `gpio_acknowledge_irq()` at the end, which is required to clear the interrupt flag in hardware; skipping it leaves the pin "stuck" re-firing the same interrupt.

## Results

The LED sequence runs continuously as a roulette. Pressing the win button while the middle LED (GPIO4) is lit switches the board into a synchronized blink of all 5 LEDs; pressing it again while blinking resets the game and the roulette starts over from the first LED. Pressing the win button while any other LED is lit has no visible effect. The speed button reliably cycles the roulette through slow, medium, and fast steps on each press, independent of whether the game is currently won or running.

<!-- SPACE FOR VIDEO -->
<!-- LED Roulette -->
<iframe width="640" height="360"
  src=""
  title="LED Roulette"
  frameborder="0"
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
  allowfullscreen>
</iframe>

## Conclusions

Moving from polling (as in the earlier moving-LED lab, which used flags `f1`/`f2` to detect a single press inside the main loop) to interrupts meant the main loop no longer has to check button state at all — it just reacts to `gane` and `vel` whenever they get updated. This also made a subtlety obvious: because only one callback can be registered per core, a router function is needed to dispatch by pin, and every shared variable between the ISR and `main()` has to be `volatile` or the change may never be seen outside the interrupt. The activity also surfaced a limitation not yet solved here — without software debounce, a single physical press can occasionally register as more than one edge, which would need a time-based filter (e.g. comparing against `time_us_32()`) to fix properly.