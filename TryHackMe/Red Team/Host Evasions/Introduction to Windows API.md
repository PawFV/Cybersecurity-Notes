---
tags: [red-team, tryhackme, room, metadata]
created: 2026-04-23
source: TryHackMe - Host Evasions
room_code: windowsapi
related:
  - "[[Host Evasions]]"
---

# Windows.h — Headers incluidos

> `Windows.h` es la **caja de herramientas maestra** de Windows para C/C++. Vos la incluís una sola vez y ella arrastra un montón de headers más chicos, cada uno especializado en una parte del sistema. Pensalo como abrir la caja de fusibles de una casa: tocás una puerta y aparecen todos los circuitos.

---

## Headers del C estándar (los "básicos")

### `<ctype.h>`
Clasificación de caracteres. Pregunta cosas tipo *"¿esto es una letra? ¿un número? ¿un espacio?"*.
**Metáfora:** el portero del boliche mirando si la letra es mayor de edad.

### `<stdarg.h>`
Soporte para funciones con número variable de argumentos (tipo `printf`).
**Metáfora:** una valija elástica: no sabés cuánta ropa le vas a meter, pero entra.

### `<string.h>`
Manipulación de strings y buffers en memoria (`strcpy`, `memcpy`, etc.).
**Metáfora:** las tijeras y el pegamento para cortar y pegar texto crudo.

---

## Headers núcleo de Windows (el "kernel" de la API)

### `<basetsd.h>`
Tipos de datos base portables entre 32/64 bits (`INT_PTR`, `DWORD_PTR`, etc.).
**Metáfora:** los adaptadores universales de enchufe — el mismo tipo anda en cualquier país.

### `<guiddef.h>`
Define el **GUID** — identificador único global de 128 bits.
**Metáfora:** el DNI cósmico. Ningún otro objeto en el universo Windows tiene el mismo.

### `<imm.h>`
Input Method Editor — escribir idiomas complejos (chino, japonés, coreano).
**Metáfora:** el traductor que convierte tus tecleos latinos en kanji.

### `<winbase.h>`
Servicios del kernel: `CreateProcess`, manejo de archivos, memoria, sincronización (`kernel32.dll` + `advapi32.dll`).
**Metáfora:** la sala de máquinas del barco. Acá se crea, se mata y se coordina todo proceso.

### `<wincon.h>`
Servicios de consola (la ventana negra de cmd).
**Metáfora:** el viejo terminal VT100 empotrado dentro de Windows.

### `<windef.h>`
Macros y tipos básicos (`BOOL`, `HANDLE`, `LPVOID`).
**Metáfora:** el diccionario de símbolos que todos los demás headers usan para hablar.

### `<winerror.h>`
Códigos de error (los famosos `ERROR_ACCESS_DENIED`, `0x5`, etc.).
**Metáfora:** el manual de "qué significa cada luz roja del tablero del auto".

### `<wingdi.h>`
**Graphics Device Interface** — dibujar líneas, texto, imágenes en pantalla o