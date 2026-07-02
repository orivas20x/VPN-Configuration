#VPN Site-to-Site IKEv1 Basada en Enrutamiento

---

##1. Objetivo de la VPN

El objetivo de este laboratorio es implementar una VPN Site-to-Site usando **IPSec IKEv1** en modo **Route-Based**. A diferencia de la basada en políticas, aquí se crea una interfaz de túnel virtual (Tunnel0) y se define una ruta estática para dirigir el tráfico, lo que ofrece mayor flexibilidad y escalabilidad.

---

##2. Topología, Interfaces y Direccionamiento IP

| Dispositivo | Interfaz | Dirección IP / Máscara | Gateway / Peer |
|---|---|---|---|
| ISP | f0/0 (Hacia A) | 10.0.0.1/30 | - |
| ISP | f0/1 (Hacia B) | 10.0.0.5/30 | - |
| Peer_A (HQ) | f0/0 (WAN) | 10.0.0.2/30 | 10.0.0.1 |
| Peer_A (HQ) | f0/1 (LAN) | 192.168.24.1/24 | - |
| Peer_A (HQ) | Tunnel0 | 172.16.24.1/30 | - |
| Peer_B (BR) | f0/1 (WAN) | 10.0.0.6/30 | 10.0.0.5 |
| Peer_B (BR) | f0/0 (LAN) | 192.168.21.1/24 | - |
| Peer_B (BR) | Tunnel0 | 172.16.24.2/30 | - |
| PC_A | vpc | 192.168.24.10/24 | 192.168.24.1 |
| PC_B | vpc | 192.168.21.10/24 | 192.168.21.1 |

### foto de la topología

<img width="432" height="409" alt="Screenshot 2026-07-02 171023" src="https://github.com/user-attachments/assets/46f5d1ff-d7c4-4f11-b0c9-3956e8d0d7b0" />


---

## 3. Parámetros Técnicos (Configuración de Criptografía)

### Fase 1 (ISAKMP)

- **Algoritmo de Cifrado:** AES-128.
- **Algoritmo de Hash:** SHA (SHA-1).
- **Grupo Diffie-Hellman:** Grupo 2 (1024-bit).
- **Autenticación:** Pre-Shared Key: *cisco2421*.

### Fase 2 (IPSec)

- **Transform-set:** ESP-AES y ESP-SHA-HMAC.
- **Modo:** Tunnel Mode.
- **Perfil IPSec:** VPNPROFILE.
- **Interfaz de Túnel:** Tunnel0 (172.16.24.0/30).
- **Ruta estática Peer_A:** 192.168.21.0/24 via 172.16.24.2.
- **Ruta estática Peer_B:** 192.168.24.0/24 via 172.16.24.1.

### Prueba de Conectividad (Ping)

<img width="668" height="484" alt="Screenshot 2026-07-02 171116" src="https://github.com/user-attachments/assets/99197382-f8aa-48e0-b715-8bf2cd3bba09" />


### Verificación de Túnel Activo

<img width="665" height="485" alt="Screenshot 2026-07-02 171146" src="https://github.com/user-attachments/assets/04834e87-5459-4920-bb11-ca6f1dea6ae4" />


<img width="668" height="487" alt="Screenshot 2026-07-02 171215" src="https://github.com/user-attachments/assets/010dbbe6-d269-4aa1-8e9e-17e558f7bdbf" />


---

##4. Scripts de Configuración (Cisco CLI)

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

! --- Interfaz Tunnel0 ---
interface Tunnel0
 ip address 172.16.24.1 255.255.255.252
 tunnel source FastEthernet0/0
 tunnel destination 10.0.0.6
 tunnel mode ipsec ipv4
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

! --- Interfaz Tunnel0 ---
interface Tunnel0
 ip address 172.16.24.2 255.255.255.252
 tunnel source FastEthernet0/1
 tunnel destination 10.0.0.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile VPNPROFILE
exit

! --- Ruta estática hacia LAN remota ---
ip route 192.168.24.0 255.255.255.0 172.16.24.1

end
write memory
```



