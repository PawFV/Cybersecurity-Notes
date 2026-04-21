---
tags: [red-team, persistence, windows]
created: 2026-04-20
source: TryHackMe - Windows Local Persistence
related:
  - "[[Tampering With Unprivileged Accounts]]"
  - "[[Persisting Through Existing Services]]"
---
La persistencia es el arte de plantar backdoors en un sistema sin ser detectado el mayor tiempo posible. En este room se cubrieron los metodos principales sobre componentes de Windows:

| Tecnica                              | Componente                                           |
| ------------------------------------ | ---------------------------------------------------- |
| Tampering With Unprivileged Accounts | Grupos, privilegios, RID hijacking                   |
| Backdooring Files                    | Ejecutables, shortcuts, file associations            |
| Abusing Services                     | Crear/modificar servicios de Windows                 |
| Scheduled Tasks                      | Task Scheduler + ocultar con SD                      |
| Logon Triggered                      | Startup folder, Run/RunOnce, Winlogon, logon scripts |
| Login Screen                         | Sticky Keys, Utilman                                 |
| Existing Services                    | Web shells, MSSQL triggers                           |

---

## Recursos adicionales

- [Hexacorn - Windows Persistence](https://www.hexacorn.com/blog/category/autostart-persistence/)
- [PayloadsAllTheThings - Windows Persistence](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Persistence.md)
- [Oddvar Moe - Persistence Through RunOnceEx](https://oddvar.moe/2018/03/21/persistence-using-runonceex-hidden-from-autoruns-exe/)
- [PowerUpSQL](https://github.com/NetSPI/PowerUpSQL)
