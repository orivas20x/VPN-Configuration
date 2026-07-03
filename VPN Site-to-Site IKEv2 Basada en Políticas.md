# VPN Site-to-Site IKEv2 Basada en Políticas

---

## 1. Objetivo de la VPN

El objetivo de este laboratorio es implementar una VPN Site-to-Site basada en políticas usando el protocolo **IPSec IKEv2**. IKEv2 es la evolución de IKEv1: ofrece una negociación más rápida y eficiente, mejor resistencia a ataques DoS y un esquema de autenticación más flexible. El tráfico se cifra según una ACL que define las redes LAN de origen y destino.

---

## 2. Topología, Interfaces y Direccionamiento IP

| Dispositivo | Interfaz | Dirección IP / Máscara | Gateway / Peer |
|---|---|---|---|
| ISP | f0/0 (Hacia A) | 10.0.0.1/30 | - |
| ISP | f1/0 (Hacia B) | 10.0.0.5/30 | - |
| Peer_A (HQ) | Gi0/0 (WAN) | 10.0.0.2/30 | 10.0.0.1 |
| Peer_A (HQ) | Gi0/1 (LAN) | 192.168.24.1/24 | - |
| Peer_B (BR) | Gi0/1 (WAN) | 10.0.0.6/30 | 10.0.0.5 |
| Peer_B (BR) | Gi0/0 (LAN) | 192.168.21.1/24 | - |
| PC_A | vpc | 192.168.24.10/24 | 192.168.24.1 |
| PC_B | vpc | 192.168.21.10/24 | 192.168.21.1 |

### foto de la topología

<img width="259" height="337" alt="Screenshot 2026-07-03 023324" src="https://github.com/user-attachments/assets/251b4025-1ea3-4dcd-b547-428cf891dd91" />


> **Nota:** El ISP usa c3725 (Dynamips) mientras que los peers usan imágenes **IOSv 15.7**, requeridas porque IKEv2 no está soportado en las imágenes IOS 12.4 de Dynamips.

---

## 3. Parámetros Técnicos (Configuración de Criptografía)

### Fase 1 (IKEv2)

- **Algoritmo de Cifrado:** AES-CBC-128.
- **Algoritmo de Integridad:** SHA-1.
- **Grupo Diffie-Hellman:** Grupo 2 (1024-bit).
- **Autenticación:** Pre-Shared Key (PSK): *cisco2421*.
- **Componentes:** Proposal, Policy, Keyring y Profile.

### Fase 2 (IPSec)

- **Transform-set:** ESP-AES y ESP-SHA-HMAC.
- **Modo:** Tunnel Mode.
- **ACL de Política:** 100 (192.168.24.0/24 ↔ 192.168.21.0/24).
- **Crypto Map:** VPNMAP con `set ikev2-profile`.

### Prueba de Conectividad (Ping)

<img width="669" height="494" alt="Screenshot 2026-07-03 023059" src="https://github.com/user-attachments/assets/9e176fe2-40d8-465a-9cd3-0c63721e77d5" />


### Verificación de Túnel Activo

<img width="668" height="496" alt="Screenshot 2026-07-03 023144" src="https://github.com/user-attachments/assets/193a3087-031f-4d45-834d-409ecfef3cb1" />


<img width="672" height="493" alt="Screenshot 2026-07-03 023212" src="https://github.com/user-attachments/assets/beb466e0-3f23-4f28-9143-95cd3d23b2c6" />


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

! --- IKEv2 Proposal ---
crypto ikev2 proposal PROP-IKEv2
 encryption aes-cbc-128
 integrity sha1
 group 2
exit

! --- IKEv2 Policy ---
crypto ikev2 policy POL-IKEv2
 proposal PROP-IKEv2
exit

! --- IKEv2 Keyring ---
crypto ikev2 keyring KR-IKEv2
 peer PEER_B
  address 10.0.0.6
  pre-shared-key cisco2421
  exit
exit

! --- IKEv2 Profile ---
crypto ikev2 profile PROF-IKEv2
 match identity remote address 10.0.0.6 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KR-IKEv2
exit

! --- Transform-set + Crypto Map ---
crypto ipsec transform-set TS esp-aes esp-sha-hmac
 mode tunnel
exit
access-list 100 permit ip 192.168.24.0 0.0.0.255 192.168.21.0 0.0.0.255
crypto map VPNMAP 10 ipsec-isakmp
 set peer 10.0.0.6
 set transform-set TS
 set ikev2-profile PROF-IKEv2
 match address 100
exit
interface GigabitEthernet0/0
 crypto map VPNMAP
exit

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

! --- IKEv2 Proposal ---
crypto ikev2 proposal PROP-IKEv2
 encryption aes-cbc-128
 integrity sha1
 group 2
exit

! --- IKEv2 Policy ---
crypto ikev2 policy POL-IKEv2
 proposal PROP-IKEv2
exit

! --- IKEv2 Keyring ---
crypto ikev2 keyring KR-IKEv2
 peer PEER_A
  address 10.0.0.2
  pre-shared-key cisco2421
  exit
exit

! --- IKEv2 Profile ---
crypto ikev2 profile PROF-IKEv2
 match identity remote address 10.0.0.2 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KR-IKEv2
exit

! --- Transform-set + Crypto Map ---
crypto ipsec transform-set TS esp-aes esp-sha-hmac
 mode tunnel
exit
access-list 100 permit ip 192.168.21.0 0.0.0.255 192.168.24.0 0.0.0.255
crypto map VPNMAP 10 ipsec-isakmp
 set peer 10.0.0.2
 set transform-set TS
 set ikev2-profile PROF-IKEv2
 match address 100
exit
interface GigabitEthernet0/1
 crypto map VPNMAP
exit

end
write memory
```

