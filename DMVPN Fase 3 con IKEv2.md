# Práctica #8 — DMVPN Fase 3 con IKEv2

**Estudiante:** Oriel Vásquez
**Matrícula:** 2024-2421
**Asignatura:** Seguridad de Redes — ITLA
**Fecha:** 4 de julio de 2026

---

## 1. Objetivo

Implementar una red **DMVPN Fase 3** usando **IPSec con IKEv2** sobre túneles **mGRE**. La Fase 3 usa **NHRP redirect** (HUB) y **NHRP shortcut** (spokes) para resumir la tabla de enrutamiento de los spokes y crear atajos directos spoke-a-spoke bajo demanda con nexthop-override.

---

## 2. Topología

<img width="419" height="331" alt="Screenshot 2026-07-03 225633" src="https://github.com/user-attachments/assets/ca84ab91-fa54-45b7-bc64-6160f9810b11" />


| Dispositivo | Rol | Interfaz WAN | Interfaz LAN |
|---|---|---|---|
| **R1** | ISP / WAN cloud (c3725) | f1/0, f2/0, f0/0 | — |
| **IOSvR-2** | HUB (NHS + redirect) | Gi0/1 | Gi0/0 → PC3 |
| **IOSvR-3** | SPOKE1 (shortcut) | Gi0/0 | Gi0/1 → PC2 |
| **IOSvR-1** | SPOKE2 (shortcut) | Gi0/0 | Gi0/1 → PC1 |

> HUB y SPOKES deben ser **IOSv** (IKEv2 no está soportado en c3725). El ISP sí puede ser c3725.

---

## 3. Direccionamiento IP (matrícula 2421)

###  WAN / NBMA — prefijo `24.21.x.y`

| Enlace | R1 | Router remoto |
|---|---|---|
| R1 ↔ HUB | 24.21.1.1 /30 | 24.21.1.2 /30 |
| R1 ↔ SPOKE1 | 24.21.2.1 /30 | 24.21.2.2 /30 |
| R1 ↔ SPOKE2 | 24.21.3.1 /30 | 24.21.3.2 /30 |

###  Túnel mGRE — `10.24.21.0/24`

| Router | Tunnel0 |
|---|---|
| HUB | 10.24.21.1 |
| SPOKE1 | 10.24.21.2 |
| SPOKE2 | 10.24.21.3 |

###  LAN — host `.21`

| Red | Gateway | PC |
|---|---|---|
| HUB | 192.168.10.1 | PC3 → 192.168.10.21 |
| SPOKE1 | 192.168.20.1 | PC2 → 192.168.20.21 |
| SPOKE2 | 192.168.30.1 | PC1 → 192.168.30.21 |

---

##  4. Parámetros de Criptografía

### IKEv2
| Parámetro | Valor |
|---|---|
| Proposal | PROP-IKEv2 |
| Cifrado | AES-CBC-256 |
| Integridad | SHA256 |
| Grupo DH | 14 (2048-bit) |
| Autenticación | Pre-Shared Key |
| Clave | `cisco2421` |

### IPSec + mGRE (Fase 3)
| Parámetro | Valor |
|---|---|
| Transform-set | TS (ESP-AES + ESP-SHA-HMAC) |
| Modo IPSec | **Transport** |
| Perfil | DMVPN (ikev2-profile) |
| Túnel | Tunnel0 (`gre multipoint`) |
| HUB | `ip nhrp redirect` |
| SPOKES | `ip nhrp shortcut` |
| Enrutamiento | EIGRP AS 1 + ruta sumaria /16 |

>  Fase 3 = `ip nhrp redirect` (HUB) + `ip nhrp shortcut` (spokes) + `ip summary-address eigrp 1 192.168.0.0 255.255.0.0` en el HUB.

---

##  5. Scripts de Configuración

###  R1 (ISP)
```cisco
enable
configure terminal
hostname R1
ip cef
interface FastEthernet1/0
 ip address 24.21.1.1 255.255.255.252
 no shutdown
exit
interface FastEthernet2/0
 ip address 24.21.2.1 255.255.255.252
 no shutdown
exit
interface FastEthernet0/0
 ip address 24.21.3.1 255.255.255.252
 no shutdown
exit
end
write memory
```

###  HUB (IOSvR-2)
```cisco
enable
configure terminal
hostname HUB
ip cef
crypto ikev2 proposal PROP-IKEv2
 encryption aes-cbc-256
 integrity sha256
 group 14
exit
crypto ikev2 policy POL-IKEv2
 proposal PROP-IKEv2
exit
crypto ikev2 keyring KR-IKEv2
 peer ANY
  address 0.0.0.0 0.0.0.0
  pre-shared-key cisco2421
 exit
exit
crypto ikev2 profile PROF-IKEv2
 match identity remote address 0.0.0.0
 authentication local pre-share
 authentication remote pre-share
 keyring local KR-IKEv2
exit
crypto ipsec transform-set TS esp-aes esp-sha-hmac
 mode transport
exit
crypto ipsec profile DMVPN
 set transform-set TS
 set ikev2-profile PROF-IKEv2
exit
interface GigabitEthernet0/1
 ip address 24.21.1.2 255.255.255.252
 no shutdown
exit
interface Tunnel0
 ip address 10.24.21.1 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp authentication cisco2421
 ip nhrp network-id 1
 ip nhrp map multicast dynamic
 ip nhrp redirect
 ip summary-address eigrp 1 192.168.0.0 255.255.0.0
 no ip split-horizon eigrp 1
 no ip next-hop-self eigrp 1
 tunnel source GigabitEthernet0/1
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN
exit
interface GigabitEthernet0/0
 ip address 192.168.10.1 255.255.255.0
 no shutdown
exit
ip route 0.0.0.0 0.0.0.0 24.21.1.1
router eigrp 1
 network 10.24.21.0 0.0.0.255
 network 192.168.10.0
 no auto-summary
exit
end
write memory
```

###  SPOKE1 (IOSvR-3)
```cisco
enable
configure terminal
hostname SPOKE1
ip cef
crypto ikev2 proposal PROP-IKEv2
 encryption aes-cbc-256
 integrity sha256
 group 14
exit
crypto ikev2 policy POL-IKEv2
 proposal PROP-IKEv2
exit
crypto ikev2 keyring KR-IKEv2
 peer ANY
  address 0.0.0.0 0.0.0.0
  pre-shared-key cisco2421
 exit
exit
crypto ikev2 profile PROF-IKEv2
 match identity remote address 0.0.0.0
 authentication local pre-share
 authentication remote pre-share
 keyring local KR-IKEv2
exit
crypto ipsec transform-set TS esp-aes esp-sha-hmac
 mode transport
exit
crypto ipsec profile DMVPN
 set transform-set TS
 set ikev2-profile PROF-IKEv2
exit
interface GigabitEthernet0/0
 ip address 24.21.2.2 255.255.255.252
 no shutdown
exit
interface Tunnel0
 ip address 10.24.21.2 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp authentication cisco2421
 ip nhrp network-id 1
 ip nhrp map 10.24.21.1 24.21.1.2
 ip nhrp map multicast 24.21.1.2
 ip nhrp nhs 10.24.21.1
 ip nhrp shortcut
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN
exit
interface GigabitEthernet0/1
 ip address 192.168.20.1 255.255.255.0
 no shutdown
exit
ip route 0.0.0.0 0.0.0.0 24.21.2.1
router eigrp 1
 network 10.24.21.0 0.0.0.255
 network 192.168.20.0
 no auto-summary
exit
end
write memory
```

###  SPOKE2 (IOSvR-1)
```cisco
enable
configure terminal
hostname SPOKE2
ip cef
crypto ikev2 proposal PROP-IKEv2
 encryption aes-cbc-256
 integrity sha256
 group 14
exit
crypto ikev2 policy POL-IKEv2
 proposal PROP-IKEv2
exit
crypto ikev2 keyring KR-IKEv2
 peer ANY
  address 0.0.0.0 0.0.0.0
  pre-shared-key cisco2421
 exit
exit
crypto ikev2 profile PROF-IKEv2
 match identity remote address 0.0.0.0
 authentication local pre-share
 authentication remote pre-share
 keyring local KR-IKEv2
exit
crypto ipsec transform-set TS esp-aes esp-sha-hmac
 mode transport
exit
crypto ipsec profile DMVPN
 set transform-set TS
 set ikev2-profile PROF-IKEv2
exit
interface GigabitEthernet0/0
 ip address 24.21.3.2 255.255.255.252
 no shutdown
exit
interface Tunnel0
 ip address 10.24.21.3 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp authentication cisco2421
 ip nhrp network-id 1
 ip nhrp map 10.24.21.1 24.21.1.2
 ip nhrp map multicast 24.21.1.2
 ip nhrp nhs 10.24.21.1
 ip nhrp shortcut
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN
exit
interface GigabitEthernet0/1
 ip address 192.168.30.1 255.255.255.0
 no shutdown
exit
ip route 0.0.0.0 0.0.0.0 24.21.3.1
router eigrp 1
 network 10.24.21.0 0.0.0.255
 network 192.168.30.0
 no auto-summary
exit
end
write memory
```



---

##  6. Verificación

### 6.1 IKEv2 + EIGRP en el HUB
<img width="667" height="491" alt="Screenshot 2026-07-03 225506" src="https://github.com/user-attachments/assets/49bdec20-4422-46c0-b7ad-77a02c6ef852" />


- `show crypto ikev2 sa` → 2 SA en **READY**, AES-CBC 256, **DH Grp:14**, PSK
- `show ip eigrp neighbors` → 2 vecinos estables sobre Tu0

### 6.2 Ruta sumaria en el SPOKE (diseño Fase 3)
<img width="662" height="485" alt="Screenshot 2026-07-03 225554" src="https://github.com/user-attachments/assets/bc4e2510-4eac-4a83-aee4-4307f278ba94" />


- `show ip route eigrp` → ruta sumaria **192.168.0.0/16** vía HUB
- `show dmvpn` → entrada a SPOKE2 con atributo **DT1** (Dynamic + Route Installed)

### 6.3 🌟 Atajo NHRP con nexthop-override (Fase 3)
<img width="667" height="488" alt="Screenshot 2026-07-03 225528" src="https://github.com/user-attachments/assets/213c98eb-86de-4a9b-bf9c-677639bd1fd4" />


`show ip nhrp` en SPOKE1 muestra la entrada **192.168.30.0/24 via 10.24.21.3** (LAN de SPOKE2) con flag **router rib** apuntando directo a la NBMA de SPOKE2 (24.21.3.2). Este atajo, creado por redirect/shortcut, es lo que distingue la Fase 3.

### 6.4 Conectividad PC2 → PC1
<img width="664" height="486" alt="Screenshot 2026-07-03 225609" src="https://github.com/user-attachments/assets/e4976fb6-c4df-4cef-a345-3ff69104e7aa" />


El ping responde con **ttl=62** (un solo salto) — túnel directo spoke-a-spoke.

---

## 🧩 7. Incidencias resueltas

1. **ISP con config antigua** — el router ISP conservaba direccionamiento de prácticas previas; se reconfiguró con las IPs `24.21.x`.
2. **WAN del HUB cruzada** — la IP WAN estaba en Gi0/0 pero el cable entraba por Gi0/1; se diagnosticó con `show cdp neighbors` y se movió la IP y el `tunnel source` a Gi0/1.
3. **Reanuncio EIGRP spoke-a-spoke** — no se propagaba; se resolvió con la ruta sumaria del HUB, que además es el diseño correcto de Fase 3.

---
