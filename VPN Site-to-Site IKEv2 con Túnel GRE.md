# VPN Site-to-Site IKEv2 con Túnel GRE

---

## 1. Objetivo de la VPN

El objetivo de este laboratorio es implementar una VPN Site-to-Site combinando un **túnel GRE** con cifrado **IPSec IKEv2**. GRE encapsula el tráfico entre los sitios mientras que IPSec con IKEv2 lo cifra. Es la solución más completa: permite transportar cualquier protocolo con la seguridad y eficiencia de IKEv2.

---

## 2. Topología, Interfaces y Direccionamiento IP

| Dispositivo | Interfaz | Dirección IP / Máscara | Gateway / Peer |
|---|---|---|---|
| ISP | f0/0 (Hacia A) | 10.0.0.1/30 | - |
| ISP | f1/0 (Hacia B) | 10.0.0.5/30 | - |
| Peer_A (HQ) | Gi0/0 (WAN) | 10.0.0.2/30 | 10.0.0.1 |
| Peer_A (HQ) | Gi0/1 (LAN) | 192.168.24.1/24 | - |
| Peer_A (HQ) | Tunnel0 (GRE) | 172.16.24.1/30 | - |
| Peer_B (BR) | Gi0/1 (WAN) | 10.0.0.6/30 | 10.0.0.5 |
| Peer_B (BR) | Gi0/0 (LAN) | 192.168.21.1/24 | - |
| Peer_B (BR) | Tunnel0 (GRE) | 172.16.24.2/30 | - |
| PC_A | vpc | 192.168.24.10/24 | 192.168.24.1 |
| PC_B | vpc | 192.168.21.10/24 | 192.168.21.1 |

### foto de la topología

<img width="259" height="337" alt="Screenshot 2026-07-03 023324" src="https://github.com/user-attachments/assets/fe89b0ff-2426-48d2-a68c-35f1efbf3ece" />


> **Nota:** Los peers usan imágenes **IOSv 15.7** porque IKEv2 no está soportado en las imágenes IOS 12.4 de Dynamips.

---

## 3. Parámetros Técnicos (Configuración de Criptografía)

### Fase 1 (IKEv2)

- **Algoritmo de Cifrado:** AES-CBC-128.
- **Algoritmo de Integridad:** SHA-1.
- **Grupo Diffie-Hellman:** Grupo 2 (1024-bit).
- **Autenticación:** Pre-Shared Key (PSK): *cisco2421*.

### Fase 2 (IPSec) + GRE

- **Transform-set:** ESP-AES y ESP-SHA-HMAC.
- **Modo:** Tunnel Mode.
- **Perfil IPSec:** VPNPROFILE (con `set ikev2-profile`).
- **Modo de Túnel:** GRE over IPSec (tunnel mode gre ip).
- **Ruta estática Peer_A:** 192.168.21.0/24 via 172.16.24.2.
- **Ruta estática Peer_B:** 192.168.24.0/24 via 172.16.24.1.

### Prueba de Conectividad (Ping)

<img width="668" height="488" alt="Screenshot 2026-07-03 035046" src="https://github.com/user-attachments/assets/533a9178-fb7d-406d-b3b2-e78382ab7eff" />


### Verificación de Túnel Activo

<img width="668" height="500" alt="Screenshot 2026-07-03 035115" src="https://github.com/user-attachments/assets/a7db6d37-c3a0-45fd-88fa-32c46b1ea7df" />


<img width="668" height="499" alt="Screenshot 2026-07-03 035135" src="https://github.com/user-attachments/assets/f2194d22-0f28-4a07-ad90-0f5916149ad5" />


---

## 4. Scripts de Configuración (Cisco CLI)

### Configuración ISP (R1)

```
enable
configure terminal
hostname ISP

interface FastEthernet0/0
 ip address 10.0.0.1 255.255.255.252
 no shutdown
exit
interface FastEthernet1/0
 ip address 10.0.0.5 255.255.255.252
 no shutdown
exit

end
write memory
```

### Configuración Peer_A (R2 - IOSv)

```
enable
configure terminal
hostname Peer_A

interface GigabitEthernet0/0
 ip address 10.0.0.2 255.255.255.252
 no shutdown
exit
interface GigabitEthernet0/1
 ip address 192.168.24.1 255.255.255.0
 no shutdown
exit
ip route 0.0.0.0 0.0.0.0 10.0.0.1

! --- IKEv2 (Proposal, Policy, Keyring, Profile) ---
crypto ikev2 proposal PROP-IKEv2
 encryption aes-cbc-128
 integrity sha1
 group 2
exit
crypto ikev2 policy POL-IKEv2
 proposal PROP-IKEv2
exit
crypto ikev2 keyring KR-IKEv2
 peer PEER_B
  address 10.0.0.6
  pre-shared-key cisco2421
  exit
exit
crypto ikev2 profile PROF-IKEv2
 match identity remote address 10.0.0.6 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KR-IKEv2
exit

! --- Transform-set + Perfil IPSec ---
crypto ipsec transform-set TS esp-aes esp-sha-hmac
 mode tunnel
exit
crypto ipsec profile VPNPROFILE
 set transform-set TS
 set ikev2-profile PROF-IKEv2
exit

! --- Tunnel GRE con protección IPSec ---
interface Tunnel0
 ip address 172.16.24.1 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 10.0.0.6
 tunnel mode gre ip
 tunnel protection ipsec profile VPNPROFILE
exit
ip route 192.168.21.0 255.255.255.0 172.16.24.2

end
write memory
```

### Configuración Peer_B (R3 - IOSv)

```
enable
configure terminal
hostname Peer_B

interface GigabitEthernet0/1
 ip address 10.0.0.6 255.255.255.252
 no shutdown
exit
interface GigabitEthernet0/0
 ip address 192.168.21.1 255.255.255.0
 no shutdown
exit
ip route 0.0.0.0 0.0.0.0 10.0.0.5

! --- IKEv2 (Proposal, Policy, Keyring, Profile) ---
crypto ikev2 proposal PROP-IKEv2
 encryption aes-cbc-128
 integrity sha1
 group 2
exit
crypto ikev2 policy POL-IKEv2
 proposal PROP-IKEv2
exit
crypto ikev2 keyring KR-IKEv2
 peer PEER_A
  address 10.0.0.2
  pre-shared-key cisco2421
  exit
exit
crypto ikev2 profile PROF-IKEv2
 match identity remote address 10.0.0.2 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KR-IKEv2
exit

! --- Transform-set + Perfil IPSec ---
crypto ipsec transform-set TS esp-aes esp-sha-hmac
 mode tunnel
exit
crypto ipsec profile VPNPROFILE
 set transform-set TS
 set ikev2-profile PROF-IKEv2
exit

! --- Tunnel GRE con protección IPSec ---
interface Tunnel0
 ip address 172.16.24.2 255.255.255.252
 tunnel source GigabitEthernet0/1
 tunnel destination 10.0.0.2
 tunnel mode gre ip
 tunnel protection ipsec profile VPNPROFILE
exit
ip route 192.168.24.0 255.255.255.0 172.16.24.1

end
write memory
```
