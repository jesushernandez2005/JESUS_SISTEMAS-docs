---
title: "GPIO desde el SDK hasta los registros de hardware (RP2350)"
date: 2026-08-31
tags: [sistemas-embebidos, raspberry-pi-pico-2, gpio, registros, sio, c]
---

# GPIO desde el SDK hasta los registros de hardware (RP2350)

## Contexto de la actividad

En esta actividad se trabajó con el bloque **SIO (Single-Cycle I/O)** del RP2350 para entender qué hace realmente `gpio_put()` a nivel de hardware. Se partió de un blink hecho con el Pico SDK, se reescribió usando registros (`sio_hw->gpio_set`, `sio_hw->gpio_clr`, `sio_hw->gpio_oe_set`, etc.) y finalmente se resolvieron 4 ejercicios con 4 LEDs conectados a GPIO2–GPIO5, manipulando los registros directamente con máscaras de bits.

**Conexión de hardware:**

```
GPIO2 → LED0 → resistencia → GND
GPIO3 → LED1 → resistencia → GND
GPIO4 → LED2 → resistencia → GND
GPIO5 → LED3 → resistencia → GND
```

---

## Objetivo

Comprender cómo el SDK de Raspberry Pi Pico traduce las funciones de alto nivel (`gpio_put`, `gpio_set_dir`, etc.) en escrituras directas sobre los registros del periférico SIO, y aplicar ese conocimiento para controlar 4 LEDs mediante máscaras de bits, generando distintos patrones de animación (contador binario, luz rebotando, llenado/vaciado progresivo y llenado de afuera hacia adentro).

## Material y Componentes usados

- 1 × Raspberry Pi Pico 2 W
- 4 × LED (cualquier color)
- 4 × Resistencia de 220 Ω (limitadora de corriente)
- 1 × Protoboard
- Cables jumper macho-macho
- Cable micro-USB (para programar y alimentar la Pico)

## Diagrama del circuito

La conexión física es la misma para los 4 ejercicios; lo único que cambia es el código que controla los pines. Cada LED se conecta en serie con su resistencia entre el GPIO correspondiente y GND:

```
GPIO2 → LED0 → resistencia 220Ω → GND
GPIO3 → LED1 → resistencia 220Ω → GND
GPIO4 → LED2 → resistencia 220Ω → GND
GPIO5 → LED3 → resistencia 220Ω → GND
```

<!-- ESPACIO PARA IMAGEN DEL DIAGRAMA DEL CIRCUITO -->
![Diagrama de conexión de los 4 LEDs](conexion_leds.svg)

La pata larga (ánodo) de cada LED va hacia la resistencia/GPIO, y la pata corta (cátodo) va hacia el riel de GND.

---

## Ejercicio 1 — Contador binario de 4 bits

Los 4 LEDs representan un número binario que cuenta de `0000` a `1111` (0 a 15) y se repite.

### Video

<!-- ESPACIO PARA VIDEO -->
<video controls width="640">
  <source src="ejercicio1.mp4" type="video/mp4">
  Tu navegador no soporta el elemento de video.
</video>
### Código

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

### Lógica utilizada

`sumador` es el contador que representa el número en decimal (0 a 15). Como GPIO2 es el primer pin físico usado y no el bit 0 del registro, el valor de `sumador` no se puede escribir directamente en `gpio_set` — hay que recorrerlo 2 posiciones a la izquierda con `sumador << 2` para que sus 4 bits caigan exactamente sobre GPIO2, GPIO3, GPIO4 y GPIO5. Ese desplazamiento se guarda en `numero_binario`.

Cada vuelta del ciclo primero apaga todo el grupo con `sio_hw->gpio_clr = MASK` (para no arrastrar el patrón del número anterior) y luego enciende únicamente los bits del número actual con `sio_hw->gpio_set = numero_binario`.

`sumador++` hace avanzar la cuenta en cada vuelta, y el `if (sumador > 15) sumador = 0` la reinicia al llegar al máximo representable en 4 bits, produciendo el efecto de "da la vuelta" (0 → 15 → 0 → ...).

---

## Ejercicio 2 — Luz rebotando (bouncing light)

Un solo LED encendido que se desplaza de un extremo al otro de los 4 LEDs y regresa, en un ciclo continuo.

### Video

<!-- ESPACIO PARA VIDEO -->
<video controls width="640">
  <source src="ejercicio2.mp4" type="video/mp4">
  Tu navegador no soporta el elemento de video.
</video>
### Código

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

### Lógica utilizada

Aquí `numero` no es una posición ni un índice, sino el valor real que se enciende en los LEDs: 1 (`0001`), 2 (`0010`), 4 (`0100`), 8 (`1000`). Multiplicar por 2 mueve el único bit encendido una posición hacia la izquierda en cada paso, y dividir entre 2 lo mueve una posición hacia la derecha — es la misma idea de desplazar bits, pero expresada como una operación aritmética en vez de con `<<`/`>>`.

La bandera `subiendo` decide qué operación tocar: mientras vale 1, `numero` se va duplicando (1→2→4→8); al llegar a 8 (el extremo GPIO5), `subiendo` pasa a 0 y `numero` empieza a dividirse entre 2 (8→4→2→1); al llegar de vuelta a 1 (el extremo GPIO2), `subiendo` vuelve a 1. Ese cambio de bandera en los extremos es lo que genera el efecto de "rebote".

Como siempre, `numero << 2` alinea el bit encendido con la posición física real de GPIO2–GPIO5 antes de escribirlo con `gpio_set`.

---

## Ejercicio 3 — Animación de llenado y vaciado

Los LEDs se van encendiendo progresivamente de un lado hasta llenarse todos, y luego se van apagando progresivamente hasta vaciarse.

### Video

<!-- ESPACIO PARA VIDEO -->
<video controls width="640">
  <source src="ejercicio3.mp4" type="video/mp4">
  Tu navegador no soporta el elemento de video.
</video>
### Código

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

### Lógica utilizada

En este ejercicio `numero` ya no representa un solo bit encendido (como en el ejercicio 2), sino un **bloque de bits consecutivos** que crece o se reduce. El truco está en `numero * 2 + 1`: multiplicar por 2 recorre todos los bits una posición a la izquierda, y el `+1` enciende un bit nuevo en la posición más baja. Esto hace que en cada vuelta se "prenda un LED más" empezando por GPIO2: 0 → 1 (`0001`) → 3 (`0011`) → 7 (`0111`) → 15 (`1111`).

Cuando `numero` llega a 15 (los 4 LEDs prendidos), la bandera `subiendo` cambia a 0 y entra la operación contraria: `numero / 2`, que recorre los bits una posición a la derecha, apagando siempre el LED más a la izquierda: 15 → 7 → 3 → 1 → 0. Al llegar a 0, `subiendo` vuelve a 1 y el ciclo se repite.

Es la misma idea de rebote de los ejercicios 2 y 3 anteriores, solo que aquí la multiplicación/división no mueve un único bit, sino todo el bloque de unos a la vez.

---

## Ejercicio 4 — Llenado de afuera hacia adentro

Los LEDs se encienden empezando por los extremos hacia el centro, y luego se apagan siguiendo el mismo patrón (de afuera hacia adentro), en un ciclo continuo.

### Video

<!-- ESPACIO PARA VIDEO -->
<video controls width="640">
  <source src="ejercicio4.mp4" type="video/mp4">
  Tu navegador no soporta el elemento de video.
</video>
### Código

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

### Lógica utilizada

Este patrón (`0000 → 1001 → 1111 → 0110 → 0000`) no sigue una fórmula aritmética simple como los ejercicios 1–3: los extremos (GPIO2 y GPIO5) se encienden primero, luego se completa el centro (GPIO3 y GPIO4), y al apagar se sigue el mismo orden. No hay un único `<<`, `+1` o `*2` que genere directamente esos 4 pasos.

Por eso se arman dos máscaras fijas con `|` (OR bit a bit): `MASK_EXTREMOS` une GPIO2 y GPIO5, y `MASK_CENTRO` une GPIO3 y GPIO4. En vez de forzar los pines a un valor fijo, se usa `sio_hw->gpio_togl`, que **invierte** el estado que ya tenían los pines (si estaban en 0 pasan a 1, y viceversa). Gracias a eso, el mismo `for` de 4 pasos sirve tanto para encender como para apagar: al principio los LEDs están apagados y el toggle los prende; una vez que están todos prendidos, el mismo bloque de código los apaga en el mismo orden.

El `if (n == 0 || n == 2)` decide en qué paso del `for` le toca a los extremos y en cuál al centro: en `n = 0` y `n = 2` se aplica `MASK_EXTREMOS`, y en `n = 1` y `n = 3` se aplica `MASK_CENTRO`, alternando entre ambas máscaras en cada una de las 4 vueltas.

---

## Conclusión

Trabajar directamente con los registros del SIO permitió entender que funciones del SDK como `gpio_put()` o `gpio_set_dir()` son en realidad una capa de abstracción sobre operaciones de bits muy simples: escrituras a `gpio_set`, `gpio_clr`, `gpio_togl` y `gpio_oe_set`. Controlar 4 LEDs con una sola instrucción (usando máscaras) en lugar de manipular cada pin por separado mostró la ventaja de operar varios GPIOs de forma simultánea y eficiente.

A lo largo de los 4 ejercicios se identificaron dos estrategias distintas para generar un patrón de bits:

1. **Cuando existe una regla matemática constante** (ejercicios 1, 2 y 3): el patrón se puede calcular en cada iteración con operaciones aritméticas o de desplazamiento (`<<`, `*2`, `/2`, `+1`), evitando escribir el patrón a mano.
2. **Cuando la secuencia es arbitraria** (ejercicio 4): no existe una fórmula simple, por lo que la solución más clara es usar máscaras fijas combinadas con `gpio_togl`, aprovechando que este registro invierte el estado del pin en vez de forzarlo, lo cual permite reutilizar el mismo bloque de código tanto para encender como para apagar.

En general, la actividad reforzó la importancia de entender el hardware por debajo del SDK: saber qué bit del registro corresponde a qué pin físico es la base para poder diseñar cualquier patrón de control de GPIOs, sin depender únicamente de las funciones de alto nivel.

