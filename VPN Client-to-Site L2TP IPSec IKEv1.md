# 🔐 Práctica #9 — VPN Client-to-Site L2TP / IPSec IKEv1

**Estudiante:** Oriel Vásquez
**Matrícula:** 2024-2421
**Asignatura:** Seguridad de Redes — ITLA


---

## 📌 1. Objetivo

Implementar una VPN de acceso remoto **client-to-site** punto a multipunto usando **L2TP sobre IPSec (IKEv1)**. Un cliente remoto (Kali Linux) establece un túnel seguro hacia un router Cisco que actúa como servidor VPN, recibe una IP de un pool y accede a la LAN interna corporativa.

Dos capas: **IPSec IKEv1** (transport mode) cifra el canal, y **L2TP** encapsula una sesión PPP que autentica al usuario (MS-CHAPv2) y asigna IP.

---

## 🗺️ 2. Topología

<img width="392" height="323" alt="Screenshot 2026-07-08 195358" src="https://github.com/user-attachments/assets/6631328d-61e0-4aa0-9018-c8feff3de471" />


| Elemento | Interfaz / Segmento | IP | Rol |
|---|---|---|---|
| **Kali** (cliente) | eth0 | 24.21.1.2 /30 | Cliente VPN remoto |
| **R1** (ISP) | f0/0 → Kali | 24.21.1.1 /30 | WAN pública |
| **R1** (ISP) | f0/1 → Servidor | 24.21.2.1 /30 | WAN pública |
| **VPN-SERVER** (IOSv) | Gi0/1 (WAN) | 24.21.2.2 /30 | Endpoint IPSec/L2TP |
| **VPN-SERVER** (IOSv) | Gi0/0 (LAN) | 192.168.50.1 /24 | Gateway LAN interna |
| **PC1** (VPCS) | e0 | 192.168.50.21 /24 | Host LAN interna |
| Pool L2TP | dinámico | 192.168.100.100-150 | IPs para clientes |
| Kali (túnel) | ppp0 | 192.168.100.100 | IP recibida del pool |

---

## 🔐 3. Parámetros

### IPSec IKEv1
| Parámetro | Valor |
|---|---|
| Cifrado (IKE) | AES-128 |
| Hash | SHA-1 |
| Grupo DH | 2 (MODP-1024) |
| Autenticación | Pre-Shared Key |
| PSK | `cisco2421` |
| Transform-set | ESP-AES + ESP-SHA-HMAC |
| Modo IPSec | **Transport** |

### L2TP / PPP
| Parámetro | Valor |
|---|---|
| Protocolo | L2TP (UDP 1701) |
| Autenticación PPP | MS-CHAPv2 |
| Usuario | `oriel` |
| Contraseña | `cisco2421` |
| Pool | 192.168.100.100 - 150 |
| Cliente | Kali (strongSwan + xl2tpd) |
| Servidor | Cisco IOSv (vpdn / virtual-template) |

---

## 💻 4. Configuración

### 🔹 Servidor VPN (Cisco IOSv)
```cisco
enable
configure terminal
hostname VPN-SERVER
ip cef
aaa new-model
aaa authentication ppp default local
aaa authorization network default local
username oriel password 0 cisco2421
ip local pool L2TP-POOL 192.168.100.100 192.168.100.150
crypto isakmp policy 10
 encr aes
 hash sha
 authentication pre-share
 group 2
 lifetime 86400
exit
crypto isakmp key cisco2421 address 0.0.0.0 0.0.0.0
crypto ipsec transform-set L2TP-SET esp-aes esp-sha-hmac
 mode transport
exit
crypto dynamic-map L2TP-DYN 10
 set transform-set L2TP-SET
 set nat demux
exit
crypto map L2TP-MAP 10 ipsec-isakmp dynamic L2TP-DYN
interface GigabitEthernet0/1
 ip address 24.21.2.2 255.255.255.252
 no shutdown
 crypto map L2TP-MAP
exit
interface GigabitEthernet0/0
 ip address 192.168.50.1 255.255.255.0
 no shutdown
exit
vpdn enable
vpdn-group L2TP
 accept-dialin
  protocol l2tp
  virtual-template 1
 no l2tp tunnel authentication
exit
interface Virtual-Template1
 ip unnumbered GigabitEthernet0/0
 peer default ip address pool L2TP-POOL
 ppp authentication chap ms-chap-v2
exit
ip route 0.0.0.0 0.0.0.0 24.21.2.1
end
write memory
```

### 🔹 Router ISP (R1)
```cisco
enable
configure terminal
hostname R1
ip cef
interface FastEthernet0/0
 ip address 24.21.1.1 255.255.255.252
 no shutdown
exit
interface FastEthernet0/1
 ip address 24.21.2.1 255.255.255.252
 no shutdown
exit
end
write memory
```

### 🔹 Cliente Kali — `/etc/ipsec.conf`
```
config setup
    charondebug="all"

conn L2TP-PSK
    keyexchange=ikev1
    authby=secret
    type=transport
    left=24.21.1.2
    leftprotoport=17/1701
    right=24.21.2.2
    rightprotoport=17/1701
    ike=aes128-sha1-modp1024!
    esp=aes128-sha1!
    auto=add
```

### 🔹 Cliente Kali — `/etc/ipsec.secrets`
```
24.21.1.2 24.21.2.2 : PSK "cisco2421"
```

### 🔹 Cliente Kali — `/etc/xl2tpd/xl2tpd.conf`
```
[lac vpn-server]
lns = 24.21.2.2
ppp debug = yes
pppoptfile = /etc/ppp/options.l2tpd.client
length bit = yes
```

### 🔹 Cliente Kali — `/etc/ppp/options.l2tpd.client`
```
ipcp-accept-local
ipcp-accept-remote
refuse-eap
require-mschap-v2
noccp
noauth
mtu 1280
mru 1280
noipdefault
defaultroute
usepeerdns
connect-delay 5000
name oriel
password cisco2421
```

### 🔹 Comandos de conexión (Kali)
```bash
# Instalar (una vez)
sudo apt install -y xl2tpd    # strongSwan ya viene en Kali

# Levantar la VPN
sudo ipsec restart
sudo ipsec up L2TP-PSK
sudo systemctl restart xl2tpd
sudo bash -c 'echo "c vpn-server" > /var/run/xl2tpd/l2tp-control'
sudo ip route add 192.168.50.0/24 dev ppp0
```



---

## ✅ 5. Verificación

### 5.1 Túnel IPSec + interfaz ppp0
<img width="1579" height="1117" alt="image" src="https://github.com/user-attachments/assets/ae18c2fb-ecfa-46e9-8df1-444d5ea642ff" />


- `sudo ipsec statusall` → SA **ESTABLISHED / INSTALLED**, IKEv1, AES-CBC-128/SHA1, transport
- `ip a show ppp0` → cliente recibió **192.168.100.100** del pool, peer 192.168.50.1

### 5.2 🌟 Conectividad + traceroute a la LAN interna
<img width="1293" height="979" alt="image" src="https://github.com/user-attachments/assets/5b61364b-1f4e-425c-8ffe-88ebb97946db" />


`traceroute 192.168.50.21` → 2 saltos: servidor VPN (192.168.50.1) → host destino (192.168.50.21). El tráfico viaja por el túnel.

### 5.3 Ping cliente → PC1
<img width="1578" height="1111" alt="image" src="https://github.com/user-attachments/assets/38c51435-b1eb-43d4-9e61-5655862c782a" />


Ping desde el cliente remoto hacia PC1: **4/4 respuestas, 0% de pérdida**.

---

## 🧩 6. Incidencias resueltas

1. **Enrutamiento en el cliente** — tras levantar el túnel, el tráfico a la LAN iba por la ruta pública. Se resolvió con `ip route add 192.168.50.0/24 dev ppp0`.
2. **NO_PROPOSAL_CHOSEN en Fase 2** — el transform-set no coincidía; se forzó `esp=aes128-sha1!` explícito.
3. **strongSwan 6.x** — se verificó que el plugin `stroke` estuviera activo para usar la config clásica `/etc/ipsec.conf`.
