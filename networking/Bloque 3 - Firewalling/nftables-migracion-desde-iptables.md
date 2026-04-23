---
title: nftables y migración desde iptables
tags: [networking, linux, firewall, nftables, iptables]
aliases: [migracion a nftables, nft vs iptables]
created: 2026-04-23
difficulty: Intermedio
related:
  - "[[iptables-tablas-chains-targets]]"
  - "[[iptables-conntrack-stateful]]"
  - "[[ufw-firewalld-wrappers]]"
---

# nftables y migración desde iptables

> [!abstract] TL;DR
> - **nftables** es la capa moderna de filtrado y clasificación en Linux, sucesora de `iptables`, `ip6tables`, `arptables` y `ebtables`.
> - En 2026, para reglas nuevas, **preferís nftables**. `iptables` queda como compatibilidad, troubleshooting y legado.
> - La ventaja real no es solo sintaxis: es modelo. `sets`, `maps`, familia `inet`, actualizaciones atómicas y menor duplicación reducen errores operativos.
> - Migrar bien no es "traducir comandos" y listo; hay que validar backend, orden, persistencia, interacción con wrappers y comportamiento efectivo del ruleset.

## Concepto

`nftables` resuelve varios problemas estructurales del mundo `iptables`:

- múltiples herramientas distintas según familia o capa,
- cadenas y tablas rígidas,
- explosión de reglas repetidas,
- updates no atómicos,
- complejidad para mantener reglas grandes.

Con `nft`, el kernel expone una máquina de reglas más coherente y expresiva. Una sola herramienta (`nft`) puede manejar IPv4, IPv6, bridge e incluso combinarlas en la familia **`inet`** para reglas comunes.

Ejemplo mental:

- En `iptables`, para bloquear 40 IPs armás 40 reglas o usás extensiones complementarias.
- En `nftables`, eso suele ser un **set** limpio y atómico.

## Cómo funciona

### Diferencia de arquitectura

```ascii
Antes (legado)
--------------
iptables   ip6tables   ebtables   arptables
    \          |          |           /
     \--------- reglas separadas -----/

Ahora (moderno)
---------------
                nft
                 |
         -----------------
         |   netfilter    |
         -----------------
           |    |    |
         inet  ip  ip6  bridge
```

### Ideas clave de nftables

- **Tablas y chains** siguen existiendo, pero el modelo es más flexible.
- **Hooks** se declaran explícitamente en la chain base: `input`, `forward`, `output`, etc.
- **Priorities** permiten controlar el orden relativo frente a NAT, mangle u otros subsistemas.
- **Sets** agrupan IPs, puertos o tuplas sin duplicar reglas.
- **Maps** permiten traducir una clave a una acción o valor.
- **Transacciones atómicas** aplican todo el ruleset junto, evitando estados intermedios peligrosos.

### Ejemplo nativo mínimo

```nft
table inet filter {
  chain input {
    type filter hook input priority 0;
    policy drop;

    iif "lo" accept
    ct state established,related accept
    ct state invalid drop
    tcp dport 22 ip saddr 10.20.30.0/24 accept
    tcp dport 443 accept
  }
}
```

Eso cubre con bastante claridad lo que en `iptables` terminaba repartido entre varias líneas menos legibles.

### Compatibilidad: `iptables-nft` no es `nft` nativo

Hay tres escenarios frecuentes:

1. **`iptables-legacy`**: backend viejo del framework clásico.
2. **`iptables-nft`**: comando `iptables`, pero traduciendo a `nf_tables`.
3. **`nft` nativo**: administración directa del ruleset moderno.

El escenario 2 puede servir como puente, pero no debería confundirse con una migración real completada.

> [!warning]
> No mezcles ciegamente `iptables-legacy`, `iptables-nft` y `nft` nativo en el mismo host. Lo que parece "una sola policy" puede ser en realidad dos universos paralelos, y después nadie entiende por qué una regla "está cargada" pero no impacta.

## Comandos / configuración

Inspección básica:

```bash
# Ver ruleset completo
sudo nft list ruleset

# Ver versión
nft --version

# Ver qué backend usa iptables
iptables --version
update-alternatives --display iptables
```

Traducción asistida desde reglas legacy:

```bash
# Traducir una regla puntual
iptables-translate -A INPUT -p tcp --dport 443 -j ACCEPT

# Traducir un ruleset completo guardado
iptables-save > /tmp/fw.v4
iptables-restore-translate -f /tmp/fw.v4
```

Aplicación de un ruleset nativo:

```bash
sudo tee /etc/nftables.conf > /dev/null <<'EOF'
flush ruleset

table inet filter {
  chain input {
    type filter hook input priority 0;
    policy drop;

    iif "lo" accept
    ct state established,related accept
    ct state invalid drop
    ip protocol icmp accept
    ip6 nexthdr ipv6-icmp accept
    tcp dport { 22, 443 } accept
  }

  chain forward {
    type filter hook forward priority 0;
    policy drop;
  }

  chain output {
    type filter hook output priority 0;
    policy accept;
  }
}
EOF

sudo nft -f /etc/nftables.conf
```

Persistencia típica:

```bash
sudo systemctl enable nftables
sudo systemctl restart nftables
sudo nft list ruleset
```

> [!note]
> En 2026, si el objetivo es durabilidad operativa, auditabilidad y menor complejidad, el camino recomendado es escribir reglas nativas en `nft` y no seguir expandiendo debt sobre `iptables`.

## Troubleshooting

| Síntoma | Causa probable | Comando de diagnóstico |
|---------|----------------|------------------------|
| El ruleset parece correcto pero no pega | Estás mirando `nft` y aplicando con `iptables-legacy`, o al revés | `iptables --version`, `sudo nft list ruleset` |
| La migración tradujo sintaxis pero cambió comportamiento | Diferencias de orden, sets implícitos o módulos no equivalentes | `iptables-save`, `iptables-restore-translate -f <archivo>`, pruebas de tráfico reales |
| UFW o firewalld pisan reglas manuales | Wrapper administrando el mismo backend | `systemctl status ufw`, `systemctl status firewalld`, `sudo nft list ruleset` |
| Un servicio dejó de responder en IPv6 | Migraste solo IPv4 y omitiste familia `ip6` o `inet` | `ss -tulpn`, `sudo nft list ruleset` |
| Rollout remoto riesgoso | Cargaste cambios no validados y perdiste acceso SSH | `sudo nft -c -f /etc/nftables.conf` antes de aplicar |

> [!tip]
> `nft -c -f archivo.conf` valida sintaxis sin aplicar. Para cambios remotos por SSH, ese hábito vale oro.

## Seguridad / ofensiva

### 1. Menos superficie de error humano

Gran parte del valor de `nftables` es reducir errores operativos:

- menos duplicación IPv4/IPv6,
- listas compactas con sets,
- updates atómicos,
- mejor legibilidad de intención.

Menos error humano suele equivaler a menos exposición involuntaria.

### 2. Atomicidad y ventanas de inconsistencia

En `iptables`, un script largo podía dejar unos segundos de reglas a medio aplicar. En `nftables`, la transacción completa entra o no entra. Eso evita ventanas donde:

- aceptaste todo temporalmente,
- borraste una regla antes de cargar la reemplazante,
- te cortaste el acceso remoto a mitad de cambio.

### 3. Post-explotación y enumeración

En un host comprometido, `sudo nft list ruleset` suele revelar:

- política real para ingreso/egreso,
- sets de redes confiables,
- wrappers activos,
- redirecciones, DNAT y filtros por interfaz.

Para pivoting y OPSEC, eso vale tanto como revisar rutas y servicios.

### 4. Migración mal hecha = zona gris peligrosa

El riesgo más común no es un bug del motor sino una convivencia sucia:

- Docker agregando reglas,
- firewalld generando chains,
- admins aplicando `iptables` a mano,
- servicio `nftables` cargando aparte.

Eso crea un plano de control confuso, ideal para errores defensivos y para que un atacante entienda más rápido de lo que el equipo operativo cree.

> [!danger]
> No asumas que "como usa nft backend ya migré". Si seguís gestionando policy compleja con sintaxis `iptables`, seguís arrastrando el modelo mental viejo, solo que arriba de una capa de compatibilidad.

## Relacionado

- [[iptables-tablas-chains-targets]] (Modelo clásico que estás dejando atrás)
- [[iptables-conntrack-stateful]] (Estado y conntrack siguen importando)
- [[ufw-firewalld-wrappers]] (Herramientas que suelen terminar arriba de nftables)

## Referencias

- `man nft`
- `man iptables-translate`
- `man iptables-restore-translate`
- nftables official wiki: [https://wiki.nftables.org/](https://wiki.nftables.org/)
- Netfilter project documentation: [https://www.netfilter.org/documentation/](https://www.netfilter.org/documentation/)
