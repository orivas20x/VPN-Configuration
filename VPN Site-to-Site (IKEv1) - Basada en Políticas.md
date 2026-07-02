🔐 VPN Site-to-Site IKEv1 Basada en Políticas

🎯 1. Objetivo de la VPN
El objetivo de este laboratorio es implementar una conexión segura punto a punto entre dos sitios (Peer_A y Peer_B) utilizando el protocolo IPSec IKEv1. La arquitectura está basada en políticas, lo que significa que el tráfico es cifrado únicamente cuando coincide con la Lista de Control de Acceso (ACL) que define las redes LAN origen y destino.

🗺️ 2. Topología, Interfaces y Direccionamiento IP
DispositivoInterfazDirección IP / MáscaraGateway / PeerISPf0/0 (Hacia A)10.0.0.1/30-ISPf0/1 (Hacia B)10.0.0.5/30-Peer_A (HQ)f0/0 (WAN)10.0.0.2/3010.0.0.1Peer_A (HQ)f0/1 (LAN)192.168.24.1/24-Peer_B (BR)f0/1 (WAN)10.0.0.6/3010.0.0.5Peer_B (BR)f0/0 (LAN)192.168.21.1/24-PC_Avpc192.168.24.10/24192.168.24.1PC_Bvpc192.168.21.10/24192.168.21.1

foto de la topología

