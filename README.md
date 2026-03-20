# Practica-5-VPN-Site-to-Site-y-Client-to-Site-FortiGate-GNS3

**Asignatura:** Seguridad en Redes

**Estudiante:** Roberto Faxas

**Matrícula:** 2023-0348

**Profesor:** Jonathan Esteban Rondón

**Fecha:** Marzo 2026

**Link del video**: [https://youtu.be/ENA84PSZrQ4](https://youtu.be/ENA84PSZrQ4)

---

## Descripción General

Este proyecto implementa dos topologías de **Redes Privadas Virtuales (VPN)** utilizando **FortiGate 7.0.9** y routers **Cisco IOU** en un entorno de laboratorio **GNS3**. Se configuraron túneles IPsec para establecer comunicación segura entre redes LAN remotas, verificando la conectividad mediante ping y traceroute.

### Topologías implementadas

* **VPN Site-to-Site**: Dos FortiGates como endpoints IPsec, un router ISP simulando Internet y dos redes LAN. El tráfico entre las LANs viaja cifrado dentro del túnel ESP.
* **VPN Client-to-Site**: Un router Cisco IOU como cliente VPN dial-up y un FortiGate como servidor dinámico. El router siempre inicia la negociación IKE.

---

## Topología 1 — VPN Site-to-Site

### Configuración de Red (Site-to-Site)

| Dispositivo | Interfaz | IP / Máscara | Rol |
| --- | --- | --- | --- |
| **FortiGate 1** | port1 | 200.0.0.1 / 24 | WAN — VPN endpoint A |
| **FortiGate 1** | port2 | 192.168.1.1 / 24 | LAN A — Gateway |
| **IOU-ISP** | Ethernet0/0 | 200.0.0.254 / 24 | ISP — lado A |
| **IOU-ISP** | Ethernet0/1 | 201.0.0.254 / 24 | ISP — lado B |
| **FortiGate 2** | port1 | 201.0.0.1 / 24 | WAN — VPN endpoint B |
| **FortiGate 2** | port2 | 192.168.2.1 / 24 | LAN B — Gateway |
| **PC1 (VPCS)** | e0 | 192.168.1.10 / 24 | Host LAN A |
| **PC2 (VPCS)** | e0 | 192.168.2.10 / 24 | Host LAN B |

---

## Configuraciones Técnicas — Site-to-Site

### IOU-ISP

```
hostname IOU-ISP

interface Ethernet0/0
 ip address 200.0.0.254 255.255.255.0
 no shutdown

interface Ethernet0/1
 ip address 201.0.0.254 255.255.255.0
 no shutdown

ip route 192.168.1.0 255.255.255.0 200.0.0.1
ip route 192.168.2.0 255.255.255.0 201.0.0.1
end
write memory
```

### FortiGate 1

```
config system interface
    edit "port1"
        set mode static
        set ip 200.0.0.1 255.255.255.0
        set allowaccess ping
        set role wan
    next
    edit "port2"
        set mode static
        set ip 192.168.1.1 255.255.255.0
        set allowaccess ping https ssh
        set role lan
    next
end

config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 200.0.0.254
        set device "port1"
    next
    edit 2
        set dst 192.168.2.0 255.255.255.0
        set device "VPN-SiteB"
    next
end

config vpn ipsec phase1-interface
    edit "VPN-SiteB"
        set interface "port1"
        set ike-version 1
        set peertype any
        set net-device disable
        set proposal des-sha256
        set dhgrp 5
        set remote-gw 201.0.0.1
        set psksecret "C1av3Secreta123!"
        set dpd on-idle
        set dpd-retryinterval 10
    next
end

config vpn ipsec phase2-interface
    edit "VPN-SiteB-P2"
        set phase1name "VPN-SiteB"
        set proposal des-sha256
        set pfs disable
        set src-addr-type subnet
        set src-subnet 192.168.1.0 255.255.255.0
        set dst-addr-type subnet
        set dst-subnet 192.168.2.0 255.255.255.0
    next
end

config firewall policy
    edit 1
        set name "LAN-A_to_SiteB"
        set srcintf "port2"
        set dstintf "VPN-SiteB"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
        set logtraffic all
    next
    edit 2
        set name "SiteB_to_LAN-A"
        set srcintf "VPN-SiteB"
        set dstintf "port2"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
        set logtraffic all
    next
end
```

### FortiGate 2

```
config system interface
    edit "port1"
        set mode static
        set ip 201.0.0.1 255.255.255.0
        set allowaccess ping
        set role wan
    next
    edit "port2"
        set mode static
        set ip 192.168.2.1 255.255.255.0
        set allowaccess ping https ssh
        set role lan
    next
end

config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 201.0.0.254
        set device "port1"
    next
    edit 2
        set dst 192.168.1.0 255.255.255.0
        set device "VPN-SiteA"
    next
end

config vpn ipsec phase1-interface
    edit "VPN-SiteA"
        set interface "port1"
        set ike-version 1
        set peertype any
        set net-device disable
        set proposal des-sha256
        set dhgrp 5
        set remote-gw 200.0.0.1
        set psksecret "C1av3Secreta123!"
        set dpd on-idle
        set dpd-retryinterval 10
    next
end

config vpn ipsec phase2-interface
    edit "VPN-SiteA-P2"
        set phase1name "VPN-SiteA"
        set proposal des-sha256
        set pfs disable
        set src-addr-type subnet
        set src-subnet 192.168.2.0 255.255.255.0
        set dst-addr-type subnet
        set dst-subnet 192.168.1.0 255.255.255.0
    next
end

config firewall policy
    edit 1
        set name "LAN-B_to_SiteA"
        set srcintf "port2"
        set dstintf "VPN-SiteA"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
        set logtraffic all
    next
    edit 2
        set name "SiteA_to_LAN-B"
        set srcintf "VPN-SiteA"
        set dstintf "port2"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
        set logtraffic all
    next
end
```

### VPCS

```
PC1> ip 192.168.1.10 192.168.1.1 24
PC2> ip 192.168.2.10 192.168.2.1 24
```

---

## Ejecución y Verificación — Site-to-Site

### 1. Estado del túnel IPsec (Verificación)

* **Comando**: `get vpn ipsec tunnel summary`
* **Resultado**: `selectors(total,up): 1/1` — túnel activo y negociado correctamente.

### 2. Ping entre LANs (Verificación)

* **Comando**: `PC1> ping 192.168.2.10`
* **Resultado**: 100% de paquetes exitosos.

### 3. Traceroute entre LANs (Verificación)

* **Comando**: `PC1> trace 192.168.2.10`
* **Resultado**:
  * Hop 1: `192.168.1.1` — FortiGate 1 (LAN gateway)
  * Hop 2: `201.0.0.1` — FortiGate 2 (VPN endpoint remoto)
  * Hop 3: `192.168.2.10` — PC2 destino

---

## Topología 2 — VPN Client-to-Site

### Configuración de Red (Client-to-Site)

| Dispositivo | Interfaz | IP / Máscara | Rol |
| --- | --- | --- | --- |
| **IOU-Client** | Ethernet0/0 | 192.168.1.1 / 24 | LAN A — Gateway |
| **IOU-Client** | Ethernet0/1 | 200.0.0.1 / 24 | WAN — Cliente VPN dial-up |
| **IOU-ISP** | Ethernet0/0 | 200.0.0.254 / 24 | ISP — lado cliente |
| **IOU-ISP** | Ethernet0/1 | 201.0.0.254 / 24 | ISP — lado FortiGate |
| **FortiGate** | port1 | 201.0.0.1 / 24 | WAN — Servidor VPN |
| **FortiGate** | port2 | 192.168.2.1 / 24 | LAN B — Gateway |
| **PC1 (VPCS)** | e0 | 192.168.1.10 / 24 | Host LAN A |
| **PC2 (VPCS)** | e0 | 192.168.2.10 / 24 | Host LAN B |

---

## Configuraciones Técnicas — Client-to-Site

### IOU-Client (Router Cisco — cliente VPN)

```
hostname Router-Client

interface Ethernet0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown

interface Ethernet0/1
 ip address 200.0.0.1 255.255.255.0
 crypto map VPNMAP
 no shutdown

ip route 0.0.0.0 0.0.0.0 200.0.0.254

crypto isakmp policy 10
 encr des
 hash sha
 authentication pre-share
 group 5
 lifetime 86400

crypto isakmp key C1av3Secreta123! address 201.0.0.1

crypto ipsec transform-set TSET esp-des esp-sha-hmac
 mode tunnel

ip access-list extended VPN_ACL
 permit ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255

crypto map VPNMAP 10 ipsec-isakmp
 set peer 201.0.0.1
 set transform-set TSET
 match address VPN_ACL
end
write memory
```

### FortiGate (Servidor VPN dial-up)

```
config system interface
    edit "port1"
        set mode static
        set ip 201.0.0.1 255.255.255.0
        set allowaccess ping
        set role wan
    next
    edit "port2"
        set mode static
        set ip 192.168.2.1 255.255.255.0
        set allowaccess ping https ssh
        set role lan
    next
end

config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 201.0.0.254
        set device "port1"
    next
    edit 2
        set dst 192.168.1.0 255.255.255.0
        set device "VPN-DialUp"
    next
end

config vpn ipsec phase1-interface
    edit "VPN-DialUp"
        set type dynamic
        set interface "port1"
        set ike-version 1
        set peertype any
        set net-device disable
        set proposal des-sha1
        set dhgrp 5
        set psksecret "C1av3Secreta123!"
        set dpd on-idle
        set dpd-retryinterval 10
    next
end

config vpn ipsec phase2-interface
    edit "VPN-DialUp-P2"
        set phase1name "VPN-DialUp"
        set proposal des-sha1
        set pfs disable
        set src-addr-type subnet
        set src-subnet 192.168.2.0 255.255.255.0
        set dst-addr-type subnet
        set dst-subnet 192.168.1.0 255.255.255.0
    next
end

config firewall policy
    edit 1
        set name "LAN-B_to_Client"
        set srcintf "port2"
        set dstintf "VPN-DialUp"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
        set logtraffic all
    next
    edit 2
        set name "Client_to_LAN-B"
        set srcintf "VPN-DialUp"
        set dstintf "port2"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
        set logtraffic all
    next
end
```

---

## Ejecución y Verificación — Client-to-Site

### 1. ISAKMP SA en Router-Client (Verificación)

* **Comando**: `show crypto isakmp sa`
* **Resultado**: `QM_IDLE ACTIVE` — túnel IKE negociado correctamente.

### 2. Estado del túnel IPsec en FortiGate (Verificación)

* **Comando**: `get vpn ipsec tunnel summary`
* **Resultado**: `'VPN-DialUp_0' selectors(total,up): 1/1`

### 3. Ping entre LANs (Verificación)

* **Comando**: `PC1> ping 192.168.2.10`
* **Resultado**: 90-100% de paquetes exitosos.

### 4. Traceroute entre LANs (Verificación)

* **Comando**: `PC1> trace 192.168.2.10`
* **Resultado**:
  * Hop 1: `192.168.1.1` — IOU-Client (LAN A gateway)
  * Hop 2: `* * *` — ISP (no responde ICMP, comportamiento normal)
  * Hop 3: `192.168.2.10` — PC2 destino

---

## Requisitos

* **GNS3** con imágenes FortiGate VM KVM y Cisco IOU L3.
* **FortiGate VM** versión 7.0.9 con licencia de evaluación válida.
* **Cisco IOU L3** versión 15.4.1T.
* **VPCS** integrado en GNS3.
* Sistema operativo host compatible con KVM/QEMU.

---

**Desarrollado por:** Roberto Faxas

**Nota:** Las imágenes de FortiGate VM requieren licencia de evaluación activa para que el plano de datos procese paquetes ESP. Sin licencia válida, el tunnel summary muestra `1/1` pero `rx/tx = 0` porque el motor de crypto del kernel está bloqueado.
