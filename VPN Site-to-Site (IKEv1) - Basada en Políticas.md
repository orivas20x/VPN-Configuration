VPN Site-to-Site IKEv1 Basada en Políticas

---

##1. Objetivo de la VPN

El objetivo de este laboratorio es implementar una conexión segura punto a punto entre dos sitios (Peer_A y Peer_B) utilizando el protocolo **IPSec IKEv1**. La arquitectura está basada en políticas, lo que significa que el tráfico es cifrado únicamente cuando coincide con la Lista de Control de Acceso (ACL) que define las redes LAN origen y destino.

---

## 2. Topología, Interfaces y Direccionamiento IP

| Dispositivo | Interfaz | Dirección IP / Máscara | Gateway / Peer |
|---|---|---|---|
| ISP | f0/0 (Hacia A) | 10.0.0.1/30 | - |
| ISP | f0/1 (Hacia B) | 10.0.0.5/30 | - |
| Peer_A (HQ) | f0/0 (WAN) | 10.0.0.2/30 | 10.0.0.1 |
| Peer_A (HQ) | f0/1 (LAN) | 192.168.24.1/24 | - |
| Peer_B (BR) | f0/1 (WAN) | 10.0.0.6/30 | 10.0.0.5 |
| Peer_B (BR) | f0/0 (LAN) | 192.168.21.1/24 | - |
| PC_A | vpc | 192.168.24.10/24 | 192.168.24.1 |
| PC_B | vpc | 192.168.21.10/24 | 192.168.21.1 |

### foto de la topología

<img width="344" height="443" alt="Screenshot 2026-07-02 151444" src="https://github.com/user-attachments/assets/f0cc48d2-173d-4519-a1de-480aaa74095c" />


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
- **ACL de Política:** 100 (Permite tráfico entre 192.168.24.0/24 y 192.168.21.0/24).

### Prueba de Conectividad (Ping)

<img width="668" height="497" alt="Screenshot 2026-07-02 151138" src="https://github.com/user-attachments/assets/7ce61eec-e815-4d2e-91b5-d0be80a63e42" />


### Verificación de Túnel Activo

<img width="668" height="503" alt="Screenshot 2026-07-02 151246" src="https://github.com/user-attachments/assets/cdf36a0c-f848-4f58-8b24-af3566e0b227" />


<img width="670" height="500" alt="Screenshot 2026-07-02 151416" src="https://github.com/user-attachments/assets/0c0c2d9e-0129-4ece-a77d-4ee3c497f3d4" />
<img width="664" height="499" alt="image" src="https://github.com/user-attachments/assets/48a15513-0df4-4a94-874a-f0c0dc1ce201" />


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

! --- ACL de Tráfico Interesante ---
access-list 100 permit ip 192.168.24.0 0.0.0.255 192.168.21.0 0.0.0.255

! --- Crypto Map ---
crypto map VPNMAP 10 ipsec-isakmp
 set peer 10.0.0.6
 set transform-set TS
 match address 100
exit

! --- Aplicación en Interfaz WAN ---
interface FastEthernet0/0
 crypto map VPNMAP
exit

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

! --- ACL de Tráfico Interesante ---
access-list 100 permit ip 192.168.21.0 0.0.0.255 192.168.24.0 0.0.0.255

! --- Crypto Map ---
crypto map VPNMAP 10 ipsec-isakmp
 set peer 10.0.0.2
 set transform-set TS
 match address 100
exit

! --- Aplicación en Interfaz WAN ---
interface FastEthernet0/1
 crypto map VPNMAP
exit

end
write memory
```

---

** Oriel Vásquez |  2024-2421 |  ITLA — Seguridad de Redes**

🎥 [Video de demostración](VIDEO_LINK_AQUI)
