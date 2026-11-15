---
title: "Digital Piano with PWM (RP2350)"
date: 2026-10-26
tags: [embedded-systems, raspberry-pi-pico-2, gpio, pwm, buzzer, c, piano]
---

# Digital Piano with PWM (RP2350)

## Objective

Use the RP2350's PWM hardware to generate audio tones on a buzzer and build an 8-key digital piano covering one full octave (Do to Do). Each key plays its note while it is held down, and two extra buttons raise or lower the volume by changing the PWM duty cycle.

## Materials and Components

- 1 × Raspberry Pi Pico 2 W
- 1 × Passive buzzer (must be passive so it follows the PWM frequency)
- 10 × Push-button (8 piano keys + 2 volume buttons)
- 1 × Protoboard
- Male-to-male jumper wires
- Micro-USB cable (programming and power)

## Circuit Diagram

```
GPIO2  → Buzzer (+) ; Buzzer (−) → GND

GPIO3  → Key Do  → GND
GPIO4  → Key Re  → GND
GPIO5  → Key Mi  → GND
GPIO6  → Key Fa  → GND
GPIO7  → Key Sol → GND
GPIO8  → Key La  → GND
GPIO9  → Key Si  → GND
GPIO10 → Key Do (high) → GND

GPIO11 → Volume up   → GND
GPIO12 → Volume down → GND

(All buttons use the internal pull-up)
```

![connection diagram](PIANO.jpeg)
/// caption
Diagrama de conexión del piano digital
///

All 10 buttons are configured in a single loop with `gpio_pull_up()`, so every pin reads `1` while released and `0` while pressed. That's why the code checks for `gpio_get(pin) == 0` to detect a press.

## Source Code

```c
#include <stdio.h>
#include "pico/stdlib.h"
#include "hardware/pwm.h"

#define BUZZER 2          
#define TOP 1023          
#define PRIMERA_TECLA 3   
#define BTN_SUBIR 11      
#define BTN_BAJAR 12     


int notas[8] = {
    1046, // Do
    1175, // Re
    1318, // Mi
    1397, // Fa
    1568, // Sol
    1760, // La
    1975, // Si
    2093  // Do 
};

int main() {
    stdio_init_all();   

    //buzzer
    gpio_set_function(BUZZER, GPIO_FUNC_PWM);       
    uint slice = pwm_gpio_to_slice_num(BUZZER);    
    uint canal = pwm_gpio_to_channel(BUZZER);      
    pwm_set_wrap(slice, TOP);                       
    pwm_set_chan_level(slice, canal, 0);            
    pwm_set_enabled(slice, true);                   

    //teclas
    for (int pin = 3; pin <= 12; pin++) {
        gpio_init(pin);             
        gpio_set_dir(pin, GPIO_IN);  
        gpio_pull_up(pin);           
    }

    int volumen = 256;   

    while (true) {   

        // volumen+
        if (gpio_get(BTN_SUBIR) == 0) {          
            volumen = volumen + 64;              
            if (volumen > 512) volumen = 512;    
            printf("Volumen: %d\n", volumen);
            sleep_ms(50);                        
            while (gpio_get(BTN_SUBIR) == 0) {}  
            sleep_ms(50);                       
        }

        // volumen-
        if (gpio_get(BTN_BAJAR) == 0) {          
            volumen = volumen - 64;              
            if (volumen < 0) volumen = 0;
            printf("Volumen: %d\n", volumen);
            sleep_ms(50);
            while (gpio_get(BTN_BAJAR) == 0) {}
            sleep_ms(50);
        }

        // tecla presionada
        int tecla = -1;   
        for (int i = 0; i < 8; i++) {                  
            if (gpio_get(PRIMERA_TECLA + i) == 0) {    
                tecla = i;                             
                break;                                 
            }
        }

        
        if (tecla == -1) {
            pwm_set_chan_level(slice, canal, 0);       
        } else {

            // divisor = reloj del pico / (nota * 1024)
            float divisor = 150000000.0f / (notas[tecla] * (TOP + 1));
            pwm_set_clkdiv(slice, divisor);            
            pwm_set_chan_level(slice, canal, volumen); 
        }

        sleep_ms(10);   
    }
}
```

## How it works

GPIO2 is handed over to the PWM peripheral with `gpio_set_function(BUZZER, GPIO_FUNC_PWM)`. Each PWM slice has a counter that runs from 0 up to the wrap value `TOP` and then restarts, so one full PWM period is `TOP + 1 = 1024` counts. `pwm_gpio_to_slice_num()` and `pwm_gpio_to_channel()` find which slice and channel GPIO2 belongs to, so the code doesn't have to hard-code them.

The **pitch** of the note is controlled with the clock divider. The RP2350 runs at 150 MHz, so the output frequency is:

```
f_out = 150 000 000 / (divisor × 1024)   →   divisor = 150 000 000 / (f_note × 1024)
```

That's exactly the formula used in the code. The `notas[]` array holds one octave (Do6 to Do7), which gives these dividers:

| Key | Note | Frequency (Hz) | Divider |
|-----|------|----------------|---------|
| GPIO3  | Do  | 1046 | 140.04 |
| GPIO4  | Re  | 1175 | 124.67 |
| GPIO5  | Mi  | 1318 | 111.14 |
| GPIO6  | Fa  | 1397 | 104.86 |
| GPIO7  | Sol | 1568 | 93.42  |
| GPIO8  | La  | 1760 | 83.23  |
| GPIO9  | Si  | 1975 | 74.17  |
| GPIO10 | Do  | 2093 | 69.99  |

All dividers fall inside the hardware's valid range (1 to 256), which is why this octave was a good fit for `TOP = 1023`.

The **volume** is controlled with the duty cycle, set by `pwm_set_chan_level()`. The output stays high while the counter is below `volumen`, so the duty cycle is `volumen / 1024`. It starts at 256 (25 %) and the volume buttons move it in steps of 64 (≈6 %) between 0 (silent) and 512 (50 %). The limit is 50 % because a square wave at 50 % duty cycle is already the loudest for the buzzer; going higher would just make it quieter again.

The main loop reads everything by polling. Each volume button has a small software debounce: a `sleep_ms(50)`, then a `while` that waits until the button is released, then another `sleep_ms(50)`, so one press only changes the volume once. Then the loop scans the 8 keys from GPIO3 upward and keeps the first one it finds pressed (`break`). If no key is pressed (`tecla == -1`) the level is set to 0 to silence the buzzer; otherwise the divider is updated to that note's frequency and the level is set to the current volume.

## Results

Each key plays its note while held down and the buzzer goes silent as soon as it's released, covering the full Do–Do octave. The volume buttons change the loudness in noticeable steps and the current level is printed on the serial monitor (`Volumen: ...`). When two keys are pressed at the same time, only the lower one (the lowest GPIO) sounds, since the scan stops at the first pressed key.

### Video

<video controls width="640" height="360">
  <source src="../piano.mp4" type="video/mp4">
</video>

## Conclusions

This activity showed that PWM isn't only for dimming LEDs: by changing the clock divider the same peripheral becomes a tone generator, and the duty cycle turns into a volume control. The key relationship was `f = f_clk / (divisor × (TOP + 1))`, which made it possible to calculate the exact divider for each note instead of guessing. Unlike the roulette lab, this one went back to polling, which works fine here because the loop only waits 10 ms per pass, so the keys respond immediately. Two limitations remain: the piano is monophonic (only one PWM slice drives the buzzer, so chords aren't possible), and while a volume button is held down the `while` loop blocks, so the keys can't change notes until it is released. Using interrupts for the volume buttons, like in the roulette, would remove that block.