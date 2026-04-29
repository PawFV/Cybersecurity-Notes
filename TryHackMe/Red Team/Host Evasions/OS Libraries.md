# ASLR, Win32 API Calls y las dos formas de resolverlos

> **El problema base:** cada función de Win32 vive en memoria y necesita un puntero a su dirección. Pero **ASLR** (Address Space Layout Randomization) mezcla las direcciones en cada arranque — como si Windows cambiara la numeración de las casas cada vez que prendés la PC. Necesitás un GPS que encuentre la función, no podés memorizarte la dirección.

---

## ASLR

Imaginate un edificio donde los departamentos cambian de número cada mañana. Vos querés visitar a "Juan" (la función `MessageBox`). No podés ir a la puerta 305 porque hoy Juan quizás vive en la 712. Necesitás preguntarle al **portero** dónde vive Juan *hoy*.

Ese "portero" tiene dos implementaciones populares:
1. **Windows Header File** → para código nativo/unmanaged (C, C++).
2. **P/Invoke** → para código managed (C#, .NET).

---

## 1. Windows Header File (el loader de Windows)

**Qué es:** Microsoft liberó `windows.h` como solución directa al problema del ASLR. También se lo llama **Windows Loader**.

### Cómo funciona (alto nivel)

En **tiempo de ejecución**, el loader:
1. Detecta qué llamadas a API estás haciendo.
2. Construye una **thunk table** → una tabla de punteros a las funciones reales.
3. Cuando invocás la función, pasa por la tabla y te lleva a la dirección correcta de ese momento.

> **Metáfora:** la **thunk table** es la guía telefónica que el portero arma cada mañana. Vos pedís "Juan", el portero consulta la guía fresca del día y te redirige al depto correcto. Nunca te enterás de que la dirección cambió.

### Cómo se usa

```c
#include <windows.h>
```

Una sola línea arriba del programa **unmanaged** y ya podés llamar cualquier función Win32. Todo lo demás lo hace el loader por detrás.

---

## 2. P/Invoke (Platform Invoke)

**Qué es:** tecnología de Microsoft para llamar código **unmanaged** (DLLs nativas de Windows) desde código **managed** (.NET / C#).

> **Metáfora:** P/Invoke es el **traductor diplomático** entre dos países que hablan idiomas distintos. Tu C# (managed, con runtime niñera) quiere hablarle a una DLL de Windows (unmanaged, sin niñera). P/Invoke traduce la conversación.

### Los dos pasos

#### Paso 1 — Importar la DLL con `DllImport`

```csharp
using System;
using System.Runtime.InteropServices;

public class Program
{
    [DllImport("user32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
    ...
}
```

- `DllImport` → atributo que le dice al runtime: "cargá esta DLL".
- `CharSet.Unicode` → usar variantes `-W` de las funciones.
- `SetLastError = true` → guarda el código de error de Windows para poder consultarlo con `Marshal.GetLastWin32Error()`.
- **No va punto y coma** todavía: la declaración no está completa.

#### Paso 2 — Declarar el método como `extern`

```csharp
private static extern int MessageBox(IntPtr hWnd, string lpText, string lpCaption, uint uType);
```

- `extern` → le avisa al runtime: "el cuerpo de esta función NO está acá, está en la DLL que importé arriba".
- La firma (tipos y parámetros) debe coincidir con la función nativa.

### Resultado

Ahora podés invocar `MessageBox(...)` como si fuera un método C# cualquiera — pero por debajo estás llamando a la API de Win32.

---

## Comparación rápida

| Aspecto | `windows.h` | P/Invoke |
|---|---|---|
| Lenguaje | C / C++ (unmanaged) | C# / .NET (managed) |
| Mecanismo | Loader + thunk table | `DllImport` + `extern` |
| Resolución de ASLR | Automática en tiempo de ejecución | Delegada al runtime .NET |
| Esfuerzo del dev | Incluís un header y listo | Declarás cada función que querés usar |

---

## Nota táctica (Red Team)

- **P/Invoke** es **el pan de cada día** del maldev en C#. Todo payload que toque Win32 desde .NET (process injection, unhooking, token manipulation) pasa por acá. EDRs lo saben y monitorean imports sospechosos.
- **`windows.h`** es el camino nativo C/C++ — menos telemetría de .NET, pero requiere más cuidado con tipos y linking.
- **Contramedida AV/EDR común:** firmas sobre combinaciones de `DllImport` + APIs jugosas (`VirtualAllocEx`, `WriteProcessMemory`, `CreateRemoteThread`). Por eso se usa **Dynamic P/Invoke** o **D/Invoke** para evadir esa telemetría — resolvés los punteros manualmente sin dejar imports estáticos visibles.