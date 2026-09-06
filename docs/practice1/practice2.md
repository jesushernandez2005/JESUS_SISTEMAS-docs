---
title: "Botones y compuertas lógicas con GPIO (RP2350)"
date: 2026-09-06
tags: [sistemas-embebidos, raspberry-pi-pico-2, gpio, registros, sio, c, compuertas-logicas]
---

# Botones y compuertas lógicas con GPIO (RP2350)

## Objective

Aplicar el manejo de GPIO como entradas digitales (botones) para implementar compuertas lógicas básicas (AND, OR, XOR) en hardware, leyendo el estado de los pines mediante el registro `sio_hw->gpio_in` y máscaras de bits. Además, usar esas mismas entradas para controlar de forma dinámica la posición de un LED encendido dentro de un arreglo de 4 LEDs.

## Materials and Components

- 1 × Raspberry Pi Pico 2 W
- 1 × LED (para los ejercicios de compuertas AND, OR y XOR)
- 4 × LED (para el ejercicio de mover el LED con botones)
- 5 × Resistencia de 220 Ω (limitadora de corriente para cada LED)
- 2 × Botón (push-button)
- 1 × Protoboard
- Cables jumper macho-macho
- Cable micro-USB (para programar y alimentar la Pico)

## Circuit Diagram

**Para los ejercicios AND, OR y XOR** (un solo LED de salida, dos botones de entrada):

```
GPIO2 → LED → resistencia 220Ω → GND
GPIO6 → Botón A → GND     (pull-down interno activado)
GPIO7 → Botón B → GND     (pull-down interno activado)
```

**Para el ejercicio de mover el LED** (4 LEDs de salida, dos botones de entrada):

```
GPIO2 → LED0 → resistencia 220Ω → GND
GPIO3 → LED1 → resistencia 220Ω → GND
GPIO4 → LED2 → resistencia 220Ω → GND
GPIO5 → LED3 → resistencia 220Ω → GND
GPIO6 → Botón izquierdo → GND   (pull-down interno)
GPIO7 → Botón derecho   → GND   (pull-down interno)
```

<!-- ESPACIO PARA IMAGEN DEL DIAGRAMA DEL CIRCUITO -->

En ambos casos, los botones no necesitan resistencia externa: se activa la resistencia **pull-down interna** del propio microcontrolador (`gpio_pull_down()`), de modo que el pin lee `0` cuando el botón no está presionado y `1` cuando sí lo está.

---

## Compuerta AND

El LED se enciende únicamente cuando **los dos botones** están presionados al mismo tiempo.

### Source Code

```c
#include "pico/stdlib.h"
#include "hardware/structs/sio.h"

#define LED     2
#define BOTON1  6
#define BOTON2  7

int main(void) {

    const uint32_t MASK_LED = 1u << LED;
    const uint32_t MASK_BOTONES = (1u << BOTON1) | (1u << BOTON2);

    gpio_init(LED);
    gpio_init(BOTON1);
    gpio_init(BOTON2);

    sio_hw->gpio_oe_set = MASK_LED;
    sio_hw->gpio_oe_clr = MASK_BOTONES;

    gpio_pull_down(BOTON1);
    gpio_pull_down(BOTON2);

    while (true) {

        uint32_t entradas = sio_hw->gpio_in;

        // AND: solo es verdadero si LOS DOS bits de la mascara estan en 1
        int resultado = (entradas & MASK_BOTONES) == MASK_BOTONES;

        if (resultado) {
            sio_hw->gpio_set = MASK_LED;
        } else {
            sio_hw->gpio_clr = MASK_LED;
        }
    }
}
```

### System Operation

`sio_hw->gpio_in` contiene el estado de todos los pines de entrada del chip en un solo número de 32 bits. La máscara `MASK_BOTONES` une los bits de GPIO6 y GPIO7 con un OR (`|`). Al aplicar `entradas & MASK_BOTONES`, se filtra el registro dejando solo esos dos bits; si el resultado es exactamente igual a `MASK_BOTONES`, significa que **ambos** bits estaban en 1, es decir, que los dos botones están presionados a la vez. Este es el comportamiento de una compuerta AND aplicado directamente sobre el registro de entradas.

| Botón A | Botón B | LED (A AND B) |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | 0 |
| 1 | 0 | 0 |
| 1 | 1 | **1** |

### Results

El LED permanece apagado si no se presiona ningún botón, o si solo se presiona uno de los dos. Únicamente se enciende en el instante en que ambos botones se mantienen presionados de forma simultánea, apagándose de inmediato al soltar cualquiera de los dos.

### Conclusions

Implementar una compuerta AND por software demostró que las operaciones lógicas de hardware pueden replicarse leyendo y comparando bits de un registro, sin necesidad de una compuerta física. Comparar el resultado del filtro contra la máscara completa (`== MASK_BOTONES`) resultó más compacto que extraer cada botón por separado y compararlos con el operador lógico `&&`.

---

## Compuerta OR

El LED se enciende si **al menos uno** de los dos botones está presionado.

### Source Code

```c
#include "pico/stdlib.h"
#include "hardware/structs/sio.h"

#define LED     2
#define BOTON1  6
#define BOTON2  7

int main(void) {

    const uint32_t MASK_LED = 1u << LED;
    const uint32_t MASK_BOTONES = (1u << BOTON1) | (1u << BOTON2);

    gpio_init(LED);
    gpio_init(BOTON1);
    gpio_init(BOTON2);

    sio_hw->gpio_oe_set = MASK_LED;
    sio_hw->gpio_oe_clr = MASK_BOTONES;

    gpio_pull_down(BOTON1);
    gpio_pull_down(BOTON2);

    while (true) {

        uint32_t entradas = sio_hw->gpio_in;

        // OR: es verdadero si CUALQUIERA de los dos bits esta en 1
        int resultado = (entradas & MASK_BOTONES) != 0;

        if (resultado) {
            sio_hw->gpio_set = MASK_LED;
        } else {
            sio_hw->gpio_clr = MASK_LED;
        }
    }
}
```

### System Operation

La diferencia con la versión AND está únicamente en la comparación final: en vez de exigir que el resultado del filtro sea igual a la máscara completa, basta con que sea **distinto de cero** (`!= 0`). Esto es cierto si al menos uno de los dos bits de botón está encendido, sin importar el otro — que es justamente la definición de la compuerta OR.

| Botón A | Botón B | LED (A OR B) |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | **1** |
| 1 | 0 | **1** |
| 1 | 1 | **1** |

### Results

El LED se enciende con presionar cualquiera de los dos botones, individualmente o al mismo tiempo, y solo permanece apagado cuando ninguno de los dos está presionado.

### Conclusions

El cambio de una sola condición (`== MASK_BOTONES` por `!= 0`) fue suficiente para pasar de una compuerta AND a una OR, lo cual evidenció que ambas comparten la misma estructura de lectura y filtrado de bits — la diferencia lógica entre ambas compuertas se reduce a una sola condición de comparación.

---

## Compuerta XOR

El LED se enciende solo cuando **uno de los dos botones** está presionado, pero no ambos.

### Source Code

```c
#include "pico/stdlib.h"
#include "hardware/structs/sio.h"

#define LED     2
#define BOTON1  6
#define BOTON2  7

int main(void) {

    const uint32_t MASK_LED = 1u << LED;
    const uint32_t MASK_BOTONES = (1u << BOTON1) | (1u << BOTON2);

    gpio_init(LED);
    gpio_init(BOTON1);
    gpio_init(BOTON2);

    sio_hw->gpio_oe_set = MASK_LED;
    sio_hw->gpio_oe_clr = MASK_BOTONES;

    gpio_pull_down(BOTON1);
    gpio_pull_down(BOTON2);

    while (true) {

        uint32_t entradas = sio_hw->gpio_in;

        int a = (entradas & (1u << BOTON1)) != 0;
        int b = (entradas & (1u << BOTON2)) != 0;

        // XOR: 1 si son diferentes (uno prendido y el otro no), 0 si son iguales
        int resultado = a ^ b;

        if (resultado) {
            sio_hw->gpio_set = MASK_LED;
        } else {
            sio_hw->gpio_clr = MASK_LED;
        }
    }
}
```

### System Operation

A diferencia del AND y el OR, aquí no basta con comparar el registro filtrado contra un solo valor, porque hay **dos** combinaciones distintas que deben encender el LED (`01` y `10`). Por eso se extrae primero cada botón por separado con `entradas & (1u << pin)` y `!= 0`, obteniendo `a` y `b` como valores limpios de 0 o 1. Luego se aplica el operador `^` (XOR bit a bit), que da 1 únicamente cuando los dos valores son diferentes entre sí.

| Botón A | Botón B | LED (A XOR B) |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | **1** |
| 1 | 0 | **1** |
| 1 | 1 | 0 |

### Results

El LED se enciende al presionar solo el botón A o solo el botón B, y se apaga tanto si no se presiona ninguno como si se presionan los dos al mismo tiempo.

### Conclusions

El XOR requirió una estrategia distinta a las dos compuertas anteriores porque no existe un único valor de comparación que capture ambas combinaciones válidas; extraer cada entrada por separado y usar el operador `^` fue la forma más simple de replicar su tabla de verdad.

---

## LED móvil controlado por botones

Dos botones (izquierda y derecha) desplazan un único LED encendido dentro de un arreglo de 4 LEDs (GPIO2–GPIO5).

### Source Code

```c
#include "pico/stdlib.h"
#include "hardware/structs/sio.h"

#define PIN_BASE 2   // GP2, GP3, GP4, GP5
#define PINS     4

#define BOTON_IZQ 6
#define BOTON_DER 7

int main(void) {

    const uint32_t MASK_LEDS = 0xF << PIN_BASE;
    const uint32_t MASK_BOTONES = (1u << BOTON_IZQ) | (1u << BOTON_DER);

    for (uint pin = PIN_BASE; pin < PIN_BASE + PINS; pin++) {
        gpio_init(pin);
    }
    gpio_init(BOTON_IZQ);
    gpio_init(BOTON_DER);

    sio_hw->gpio_oe_set = MASK_LEDS;
    sio_hw->gpio_oe_clr = MASK_BOTONES;

    gpio_pull_down(BOTON_IZQ);
    gpio_pull_down(BOTON_DER);

    int pos = 0;   // posicion del LED encendido (0=GP2 ... 3=GP5)

    while (true) {

        // dibuja el LED en su posicion actual
        uint32_t pattern = (1u << pos) << PIN_BASE;
        sio_hw->gpio_clr = MASK_LEDS;
        sio_hw->gpio_set = pattern & MASK_LEDS;

        if (gpio_get(BOTON_IZQ) && pos > 0) {
            pos--;
            while (gpio_get(BOTON_IZQ));   // espera a que sueltes el boton
        }

        if (gpio_get(BOTON_DER) && pos < 3) {
            pos++;
            while (gpio_get(BOTON_DER));   // espera a que sueltes el boton
        }
    }
}
```

### System Operation

La variable `pos` guarda cuál de los 4 LEDs está encendido (0 a 3). En cada vuelta se dibuja el LED en esa posición con `1u << pos`, alineado a GPIO2–GPIO5 con `<< PIN_BASE`, igual que en la actividad anterior de animaciones.

A diferencia de las compuertas lógicas, aquí se usa `gpio_get(pin)` (una función del SDK) en vez de leer manualmente `sio_hw->gpio_in` con máscaras, ya que resulta más simple para revisar un solo botón a la vez.

Cuando se detecta el botón izquierdo presionado (y `pos` no está ya en el extremo GPIO2), se resta 1 a `pos`; cuando se detecta el botón derecho presionado (y `pos` no está en el extremo GPIO5), se le suma 1. El `while (gpio_get(BOTON_X));` inmediatamente después bloquea el programa hasta que el botón se suelte, evitando que el LED se siga moviendo varias posiciones de golpe mientras el botón permanece presionado.

### Results

Al presionar el botón izquierdo, el LED encendido se desplaza una posición hacia GPIO2 en cada pulsación; al presionar el derecho, se desplaza hacia GPIO5. El LED se detiene en los pines extremos sin dar la vuelta ni causar comportamiento inesperado si se sigue presionando el botón correspondiente al límite.

### Conclusions

Controlar la posición de un LED con botones combinó dos ideas ya trabajadas — la máscara de un solo bit desplazado (`1u << pos`) y la lectura de entradas digitales — agregando el manejo del "rebote humano" de un botón: sin bloquear el programa hasta soltar el botón, una sola pulsación provocaría múltiples desplazamientos indeseados.