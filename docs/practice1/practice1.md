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
// ESPACIO PARA CÓDIGO — Ejercicio 1: Contador binario de 4 bits
 
```
 
### Lógica utilizada
 
<!-- ESPACIO PARA EXPLICAR LA LÓGICA -->
 
 
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
// ESPACIO PARA CÓDIGO — Ejercicio 2: Luz rebotando
 
```
 
### Lógica utilizada
 
<!-- ESPACIO PARA EXPLICAR LA LÓGICA -->
 
 
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
// ESPACIO PARA CÓDIGO — Ejercicio 3: Llenado y vaciado
 
```
 
### Lógica utilizada
 
<!-- ESPACIO PARA EXPLICAR LA LÓGICA -->
 
 
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
// ESPACIO PARA CÓDIGO — Ejercicio 4: Llenado de afuera hacia adentro
 
```
 
### Lógica utilizada
 
<!-- ESPACIO PARA EXPLICAR LA LÓGICA -->
 