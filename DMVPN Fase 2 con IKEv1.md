 DMVPN Fase 2 con IKEv1


## 📌 1. Objetivo

Implementar una red **DMVPN (Dynamic Multipoint VPN) en Fase 2** usando **IPSec IKEv1** sobre túneles **mGRE**. En Fase 2, los spokes negocian **túneles directos spoke-a-spoke** de forma dinámica vía **NHRP**, sin pasar por el HUB para el tráfico de datos.

---

## 🗺️ 2. Topología

<img width="330" height="419" alt="Screenshot 2026-07-03 172416" src="https://github.com/user-attachments/assets/346930b1-7a7d-4975-a038-fc9d1adf4da9" />


| Dispositivo | Rol | Interfaz WAN | Interfaz LAN |
|---|---|---|---|
| **R1** | ISP / WAN cloud | f1/0, f2/0, f0/0 | — |
| **IOSvR-2** | HUB (NHS) | Gi0/1 | Gi0/0 → PC3 |
| **IOSvR-3** | SPOKE1 | Gi0/0 | Gi0/1 → PC2 |
| **IOSvR-1** | SPOKE2 | Gi0/0 | Gi0/1 → PC1 |

---

## 🔢 3. Direccionamiento IP (basado en matrícula 2421)

### 🔹 WAN / NBMA — prefijo `24.21.x.y`

| Enlace | R1 | Router remoto |
|---|---|---|
| R1 ↔ HUB | 24.21.1.1 /30 | 24.21.1.2 /30 |
| R1 ↔ SPOKE1 | 24.21.2.1 /30 | 24.21.2.2 /30 |
| R1 ↔ SPOKE2 | 24.21.3.1 /30 | 24.21.3.2 /30 |

### 🔹 Túnel mGRE — `10.24.21.0/24`

| Router | Tunnel0 |
|---|---|
| HUB | 10.24.21.1 |
| SPOKE1 | 10.24.21.2 |
| SPOKE2 | 10.24.21.3 |

### 🔹 LAN — host `.21`

| Red | Gateway | PC |
|---|---|---|
| HUB | 192.168.10.1 | PC3 → 192.168.10.21 |
| SPOKE1 | 192.168.20.1 | PC2 → 192.168.20.21 |
| SPOKE2 | 192.168.30.1 | PC1 → 192.168.30.21 |

---

## 🔐 4. Parámetros de Criptografía

### Fase 1 (IKEv1 / ISAKMP)
| Parámetro | Valor |
|---|---|
| Cifrado | AES |
| Hash | SHA |
| Grupo DH | 2 (1024-bit) |
| Autenticación | Pre-Shared Key |
| Clave | `Cisco123` (address 0.0.0.0) |
| Lifetime | 86400 s |

### Fase 2 (IPSec) + mGRE
| Parámetro | Valor |
|---|---|
| Transform-set | ESP-AES + ESP-SHA-HMAC |
| Modo IPSec | **Transport** |
| Perfil | DMVPN |
| Túnel | Tunnel0 (`gre multipoint`) |
| NHRP network-id | 1 |
| Enrutamiento | EIGRP AS 1 |

> ⚠️ En el HUB son obligatorios `no ip split-horizon eigrp 1` y `no ip next-hop-self eigrp 1` para habilitar los túneles directos de Fase 2.

---

## 💻 5. Scripts de Configuración

### 🔹 R1 (ISP)
```cisco
enable
configure terminal
hostname R1
ip cef
! --- Interfaces WAN ---
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

### 🔹 HUB (IOSvR-2)
```cisco
enable
configure terminal
hostname HUB
ip cef
! --- Fase 1 IKEv1 ---
crypto isakmp policy 10
 encr aes
 hash sha
 authentication pre-share
 group 2
 lifetime 86400
exit
crypto isakmp key Cisco123 address 0.0.0.0
! --- Fase 2 IPSec ---
crypto ipsec transform-set TSET esp-aes esp-sha-hmac
 mode transport
exit
crypto ipsec profile DMVPN
 set transform-set TSET
exit
! --- Interfaz WAN ---
interface GigabitEthernet0/1
 ip address 24.21.1.2 255.255.255.252
 no shutdown
exit
! --- Tunnel mGRE (HUB) ---
interface Tunnel0
 ip address 10.24.21.1 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp authentication Cisco123
 ip nhrp network-id 1
 ip nhrp map multicast dynamic
 no ip split-horizon eigrp 1
 no ip next-hop-self eigrp 1
 tunnel source GigabitEthernet0/1
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN
exit
! --- LAN ---
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

### 🔹 SPOKE1 (IOSvR-3)
```cisco
enable
configure terminal
hostname SPOKE1
ip cef
! --- Fase 1 IKEv1 ---
crypto isakmp policy 10
 encr aes
 hash sha
 authentication pre-share
 group 2
 lifetime 86400
exit
crypto isakmp key Cisco123 address 0.0.0.0
! --- Fase 2 IPSec ---
crypto ipsec transform-set TSET esp-aes esp-sha-hmac
 mode transport
exit
crypto ipsec profile DMVPN
 set transform-set TSET
exit
! --- Interfaz WAN ---
interface GigabitEthernet0/0
 ip address 24.21.2.2 255.255.255.252
 no shutdown
exit
! --- Tunnel mGRE (SPOKE) ---
interface Tunnel0
 ip address 10.24.21.2 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp authentication Cisco123
 ip nhrp network-id 1
 ip nhrp map 10.24.21.1 24.21.1.2
 ip nhrp map multicast 24.21.1.2
 ip nhrp nhs 10.24.21.1
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN
exit
! --- LAN ---
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

### 🔹 SPOKE2 (IOSvR-1)
```cisco
enable
configure terminal
hostname SPOKE2
ip cef
! --- Fase 1 IKEv1 ---
crypto isakmp policy 10
 encr aes
 hash sha
 authentication pre-share
 group 2
 lifetime 86400
exit
crypto isakmp key Cisco123 address 0.0.0.0
! --- Fase 2 IPSec ---
crypto ipsec transform-set TSET esp-aes esp-sha-hmac
 mode transport
exit
crypto ipsec profile DMVPN
 set transform-set TSET
exit
! --- Interfaz WAN ---
interface GigabitEthernet0/0
 ip address 24.21.3.2 255.255.255.252
 no shutdown
exit
! --- Tunnel mGRE (SPOKE) ---
interface Tunnel0
 ip address 10.24.21.3 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp authentication Cisco123
 ip nhrp network-id 1
 ip nhrp map 10.24.21.1 24.21.1.2
 ip nhrp map multicast 24.21.1.2
 ip nhrp nhs 10.24.21.1
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN
exit
! --- LAN ---
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

## ✅ 6. Verificación

### 6.1 Registro NHRP + IPSec en el HUB
<img width="663" height="518" alt="Screenshot 2026-07-03 165844" src="https://github.com/user-attachments/assets/807cb733-215d-49dc-86c0-78c2a858850e" />


- `show dmvpn` → 2 peers **UP / D**
- `show crypto isakmp sa` → 2 SA en **QM_IDLE / ACTIVE**
- `show ip eigrp neighbors` → 2 vecinos sobre Tu0

### 6.2 🌟 Prueba de Fase 2 (túnel directo spoke-a-spoke)
<img width="661" height="530" alt="Screenshot 2026-07-03 170006" src="https://github.com/user-attachments/assets/9a4c057f-6f79-4c95-b682-41cd3e97f76e" />


En SPOKE1, `show dmvpn` muestra:
- Entrada al **HUB** → atributo **S** (Static, configurada)
- Entrada a **SPOKE2** → atributo **D** (Dynamic, creada automáticamente por NHRP) ← **esto demuestra Fase 2**

### 6.3 Conectividad PC2 → PC1
<img width="673" height="517" alt="Screenshot 2026-07-03 170134" src="https://github.com/user-attachments/assets/14d5652c-85af-4a2f-ba62-970ba3498dd6" />


El TTL pasa de **61** (primer paquete, vía HUB) a **62** (siguientes, túnel directo) — evidencia del túnel dinámico levantándose en tiempo real.

---

## 🧩 7. Incidencia resuelta

El enlace hacia el HUB estaba cableado en una interfaz de R1 distinta a la planificada. Se diagnosticó con `show cdp neighbors` y se corrigió reasignando las IPs de R1 al cableado físico real. Tras eso, la WAN levantó y el DMVPN se formó de inmediato.

