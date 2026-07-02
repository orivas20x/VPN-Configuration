# VPN Site-to-Site IKEv1 con Túnel GRE

---

## 1. Objetivo de la VPN

El objetivo de este laboratorio es implementar una VPN Site-to-Site combinando un **túnel GRE** con cifrado **IPSec IKEv1**. El túnel GRE encapsula el tráfico entre los dos sitios, mientras que IPSec proporciona la seguridad cifrando el contenido del túnel.

---

## 2. Topología, Interfaces y Direccionamiento IP

| Dispositivo | Interfaz | Dirección IP / Máscara | Gateway / Peer |
|---|---|---|---|
| ISP | f0/0 (Hacia A) | 10.0.0.1/30 | - |
| ISP | f0/1 (Hacia B) | 10.0.0.5/30 | - |
| Peer_A (HQ) | f0/0 (WAN) | 10.0.0.2/30 | 10.0.0.1 |
| Peer_A (HQ) | f0/1 (LAN) | 192.168.24.1/24 | - |
| Peer_A (HQ) | Tunnel0 (GRE) | 172.16.24.1/30 | - |
| Peer_B (BR) | f0/1 (WAN) | 10.0.0.6/30 | 10.0.0.5 |
| Peer_B (BR) | f0/0 (LAN) | 192.168.21.1/24 | - |
| Peer_B (BR) | Tunnel0 (GRE) | 172.16.24.2/30 | - |
| PC_A | vpc | 192.168.24.10/24 | 192.168.24.1 |
| PC_B | vpc | 192.168.21.10/24 | 192.168.21.1 |

### foto de la topología

<img width="424" height="367" alt="Screenshot 2026-07-02 185429" src="https://github.com/user-attachments/assets/d47c56a5-6ed4-494e-b196-b06359bff077" />


---

## 3. Parámetros Técnicos (Configuración de Criptografía)

### Fase 1 (ISAKMP)

- **Algoritmo de Cifrado:** AES-128.
- **Algoritmo de Hash:** SHA (SHA-1).
- **Grupo Diffie-Hellman:** Grupo 2 (1024-bit).
- **Autenticación:** Pre-Shared Key: *cisco2421*.

### Fase 2 (IPSec) + GRE

- **Transform-set:** ESP-AES y ESP-SHA-HMAC.
- **Modo:** Tunnel Mode.
- **Perfil IPSec:** VPNPROFILE.
- **Modo de Túnel:** GRE over IPSec (tunnel mode gre ip).
- **Ruta estática Peer_A:** 192.168.21.0/24 via 172.16.24.2.
- **Ruta estática Peer_B:** 192.168.24.0/24 via 172.16.24.1.

### Prueba de Conectividad (Ping)

<img width="669" height="497" alt="Screenshot 2026-07-02 185521" src="https://github.com/user-attachments/assets/b07dcb39-b55e-4acb-b23a-a84cd4503ae1" />


### Verificación de Túnel Activo

<img width="671" height="499" alt="Screenshot 2026-07-02 185353" src="https://github.com/user-attachments/assets/fc905427-a009-4716-b0c0-231e85d7b1a5" />


<img width="671" height="495" alt="Screenshot 2026-07-02 185410" src="https://github.com/user-attachments/assets/e6c067e4-737a-4842-9fe4-d7eaf69ec92c" />


---

## 4. Scripts de Configuración (Cisco CLI)

### Configuración ISP (R1)

```
enable
configure terminal
hostname ISP

! --- Interfaces ---
interface FastEthernet0/0
 ip address 10.0.0.1 255.255.255.252
 no shutdown
exit
interface FastEthernet0/1
 ip address 10.0.0.5 255.255.255.252
 no shutdown
exit

end
write memory
```

### Configuración Peer_A (R2 - Local)

```
enable
configure terminal
hostname Peer_A

! --- Interfaces ---
interface FastEthernet0/0
 ip address 10.0.0.2 255.255.255.252
 no shutdown
exit
interface FastEthernet0/1
 ip address 192.168.24.1 255.255.255.0
 no shutdown
exit

! --- Enrutamiento hacia el ISP ---
ip route 0.0.0.0 0.0.0.0 10.0.0.1

! --- Tunnel GRE ---
interface Tunnel0
 ip address 172.16.24.1 255.255.255.252
 tunnel source FastEthernet0/0
 tunnel destination 10.0.0.6
 tunnel mode gre ip
exit

! --- Configuración Fase 1 ---
crypto isakmp policy 10
 encr aes 128
 hash sha
 authentication pre-share
 group 2
exit
crypto isakmp key cisco2421 address 10.0.0.6

! --- Configuración Fase 2 ---
crypto ipsec transform-set TS esp-aes esp-sha-hmac
 mode tunnel
exit

! --- Perfil IPSec ---
crypto ipsec profile VPNPROFILE
 set transform-set TS
exit

! --- Aplicar IPSec al Tunnel GRE ---
interface Tunnel0
 tunnel protection ipsec profile VPNPROFILE
exit

! --- Ruta estática hacia LAN remota ---
ip route 192.168.21.0 255.255.255.0 172.16.24.2

end
write memory
```

### Configuración Peer_B (R3 - Remoto)

```
enable
configure terminal
hostname Peer_B

! --- Interfaces ---
interface FastEthernet0/1
 ip address 10.0.0.6 255.255.255.252
 no shutdown
exit
interface FastEthernet0/0
 ip address 192.168.21.1 255.255.255.0
 no shutdown
exit

! --- Enrutamiento hacia el ISP ---
ip route 0.0.0.0 0.0.0.0 10.0.0.5

! --- Tunnel GRE ---
interface Tunnel0
 ip address 172.16.24.2 255.255.255.252
 tunnel source FastEthernet0/1
 tunnel destination 10.0.0.2
 tunnel mode gre ip
exit

! --- Configuración Fase 1 ---
crypto isakmp policy 10
 encr aes 128
 hash sha
 authentication pre-share
 group 2
exit
crypto isakmp key cisco2421 address 10.0.0.2

! --- Configuración Fase 2 ---
crypto ipsec transform-set TS esp-aes esp-sha-hmac
 mode tunnel
exit

! --- Perfil IPSec ---
crypto ipsec profile VPNPROFILE
 set transform-set TS
exit

! --- Aplicar IPSec al Tunnel GRE ---
interface Tunnel0
 tunnel protection ipsec profile VPNPROFILE
exit

! --- Ruta estática hacia LAN remota ---
ip route 192.168.24.0 255.255.255.0 172.16.24.1

end
write memory
```
