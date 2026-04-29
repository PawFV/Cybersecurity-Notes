# Win32 API — Estructura y Organización

La **Win32 API** (más conocida como **Windows API**) tiene varios componentes dependientes que definen su estructura y organización.

## Enfoque top-down

Vamos a desglosarla de arriba hacia abajo: asumimos que la **API** es la capa más alta y los **parámetros** de una llamada específica son la capa más baja.

## Capas de la Win32 API

| Capa | Explicación |
|------|-------------|
| **API** | Término general/de alto nivel para describir cualquier llamada dentro de la estructura Win32. |
| **Header files / imports** | Definen las librerías a importar en tiempo de ejecución, mediante archivos de cabecera (`.h`) o imports de librería. Usan **punteros** para obtener la dirección de la función. |
| **Core DLLs** | Grupo de DLLs principales que definen las estructuras de llamada: `KERNEL32`, `USER32`, `ADVAPI32` (y `GDI32`). Exponen servicios de kernel y usuario que no están atados a un solo subsistema. |
| **Supplemental DLLs** | Otras DLLs que forman parte de la Windows API. Controlan subsistemas separados del SO. Hay ~36 DLLs adicionales (`NTDLL`, `COM`, `FVEAPI`, etc.). |
| **Call Structures** | Definen la llamada API en sí y los parámetros que recibe. |
| **API Calls** | La llamada concreta usada dentro de un programa; la dirección de la función se obtiene vía puntero. |
| **In/Out Parameters** | Los valores de los parámetros definidos por las Call Structures (entrada y salida). |

> [!info] Próximos pasos
> - En la **próxima task**: importación de librerías, header file principal y call structure.
> - En la **task 4**: profundizamos en las calls, cómo digerir parámetros y variantes.

---

## Preguntas

### ❓ ¿Qué header file importa y define la DLL `User32` y su estructura?

**Respuesta:** `winuser.h`

### ❓ ¿Qué header file "padre" contiene todos los demás header files hijos y core requeridos?

**Respuesta:** `windows.h`

---

## 🧠 Resumen mental (para fijar)

- `windows.h` = **el padre**. Lo incluís y te trae a todos los hijos (`winuser.h`, `wingdi.h`, `winbase.h`, etc.).
- Cada core DLL tiene su header file dedicado:
  - `KERNEL32` → `winbase.h`
  - `USER32` → `winuser.h`
  - `GDI32` → `wingdi.h`
  - `ADVAPI32` → `winreg.h` / `winsvc.h` (servicios, registry)
- Los headers **declaran** las funciones; las DLLs **las implementan**. El linker/loader conecta ambos via punteros en runtime.

#win32 #api #windows #cybersecurity #reversing