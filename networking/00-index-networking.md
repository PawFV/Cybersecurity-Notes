---
title: Map of Content - Networking
tags: [networking, moc, index, devsecops]
aliases: [Networking MOC, Index Networking]
created: 2026-04-23
difficulty: N/A
related:
  - "[[osi-vs-tcpip]]"
  - "[[interfaces-ip-link-addr-route]]"
---

# Networking Map of Content (MOC)

> [!abstract] TL;DR
> Índice central de apuntes técnicos de networking enfocados en DevSecOps y seguridad ofensiva (Red Team). Organizado desde fundamentos hasta capas de aplicación y diagnóstico.

## Bloque 1 — Fundamentos
- [[osi-vs-tcpip]]
- [[ethernet-y-arp]]
- [[ip-subnetting-cidr]]
- [[routing-basico-tabla-rutas]]
- [[tcp-estados-y-handshake]]
- [[udp-vs-tcp-cuando-cual]]
- [[mtu-mss-fragmentacion]]
- [[nat-snat-dnat-masquerade]]

## Bloque 2 — Linux Networking Stack
- [[interfaces-ip-link-addr-route]]
- [[iproute2-suite-ss-ip-bridge]]
- [[network-namespaces]]
- [[systemd-networkd-vs-networkmanager]]
- [[netplan]]
- [[resolv-conf-y-systemd-resolved]]
- [[bonding-bridging-vlans]]

## Bloque 3 — Firewalling
- [[iptables-tablas-chains-targets]]
- [[iptables-conntrack-stateful]]
- [[nftables-migracion-desde-iptables]]
- [[ufw-firewalld-wrappers]]
- [[ebpf-xdp-filtering-moderno]]

## Bloque 4 — DNS
- [[dns-jerarquia-resolucion]]
- [[dns-records-A-AAAA-NS-MX-TXT-SRV-CNAME]]
- [[dig-nslookup-host-drill]]
- [[dns-over-tls-https-doh-dot]]
- [[dns-caching-forwarders-dnsmasq-unbound]]
- [[dns-tunneling-iodine-dnscat2]]

## Bloque 5 — Servicios y Protocolos de Aplicación
- [[http-vs-https-tls-handshake]]
- [[ssh-arquitectura-y-forwarding]]
- [[ssh-local-remote-dynamic-forwarding]]
- [[smb-cifs-y-shares-windows]]
- [[kerberos-basico]]
- [[ldap-y-active-directory]]

## Bloque 6 — Diagnóstico y Captura
- [[tcpdump-cheatsheet-y-bpf]]
- [[wireshark-display-filters]]
- [[nmap-tecnicas-de-scan]]
- [[traceroute-mtr-pathping]]
- [[ss-netstat-lsof-procesos-vs-puertos]]
