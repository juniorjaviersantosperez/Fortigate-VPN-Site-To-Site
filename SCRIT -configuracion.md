# Script de Configuración — VPN Site-to-Site FortiGate A / FortiGate B

Este documento contiene el **script completo de comandos** ejecutado en cada equipo de la topología para llegar al resultado final documentado en `README.md`.

---

## Topología de referencia

```
VPC6 ---- FGT-A ---- R-ISP ---- FGT-B ---- VPC7
10.15.99.2   10.15.99.1/24  200.0.0.2/30  200.0.0.5/30  200.0.0.6/30  192.168.99.1/24  192.168.99.2
(eth0)       (port2)(port1)     (e0/0)        (e0/1)       (port1)(port2)       (eth0)
```

---

## 1. R-ISP (Cisco Router)

```bash
enable
configure terminal

hostname R-ISP

interface Ethernet0/0
 ip address 200.0.0.2 255.255.255.252
 no shutdown
 duplex auto
 exit

interface Ethernet0/1
 ip address 200.0.0.5 255.255.255.252
 no shutdown
 duplex auto
 exit

end
write memory
```

### Verificación
```bash
show ip interface brief
show ip route
```

---

## 2. FGT-A (FortiGate)

### 2.1 Hostname
```bash
config system global
    set hostname "FGT-A"
end
```

### 2.2 Interfaces
```bash
config system interface
    edit "port1"
        set vdom "root"
        set ip 200.0.0.1 255.255.255.252
        set allowaccess ping https ssh
        set type physical
        set alias "WAN-A"
    next
    edit "port2"
        set vdom "root"
        set ip 10.15.99.1 255.255.255.0
        set allowaccess ping https ssh
        set type physical
        set alias "LAN-A-10.15.99.0"
    next
end
```

### 2.3 Fase 1 IKE (Phase1-interface)
```bash
config vpn ipsec phase1-interface
    edit "VPN-A-B"
        set interface "port1"
        set ike-version 2
        set peertype any
        set net-device disable
        set proposal des-sha256
        set dhgrp 14
        set remote-gw 200.0.0.6
        set psksecret <CLAVE_PRECOMPARTIDA>
    next
end
```

### 2.4 Fase 2 IPsec (Phase2-interface)
```bash
config vpn ipsec phase2-interface
    edit "VPN-A-B-P2"
        set phase1name "VPN-A-B"
        set proposal des-sha256
        set dhgrp 14
        set src-subnet 10.15.99.0 255.255.255.0
        set dst-subnet 192.168.99.0 255.255.255.0
    next
end
```

### 2.5 Rutas estáticas
```bash
config router static
    edit 1
        set gateway 200.0.0.2
        set device "port1"
    next
    edit 2
        set dst 192.168.99.0 255.255.255.0
        set device "VPN-A-B"
    next
end
```

### 2.6 Políticas de firewall
```bash
config firewall policy
    edit 1
        set name "LAN-to-VPN"
        set srcintf "port2"
        set dstintf "VPN-A-B"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
    next
    edit 2
        set name "VPN-to-LAN"
        set srcintf "VPN-A-B"
        set dstintf "port2"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
    next
end
```

### Verificación
```bash
get system status
get system interface physical
show vpn ipsec phase1-interface
show vpn ipsec phase2-interface
diagnose vpn ike gateway list
diagnose vpn tunnel list
get vpn ipsec tunnel summary
get router info routing-table all
show firewall policy
```

---

## 3. FGT-B (FortiGate)

### 3.1 Hostname
```bash
config system global
    set hostname "FGT-B"
end
```

### 3.2 Interfaces
```bash
config system interface
    edit "port1"
        set vdom "root"
        set ip 200.0.0.6 255.255.255.252
        set allowaccess ping https ssh
        set type physical
        set alias "WAN-B"
    next
    edit "port2"
        set vdom "root"
        set ip 192.168.99.1 255.255.255.0
        set allowaccess ping https ssh
        set type physical
        set alias "LAN-B-192.168.99.0"
    next
end
```

### 3.3 Fase 1 IKE (Phase1-interface)
```bash
config vpn ipsec phase1-interface
    edit "VPN-B-A"
        set interface "port1"
        set ike-version 2
        set peertype any
        set net-device disable
        set proposal des-sha256
        set dhgrp 14
        set remote-gw 200.0.0.1
        set psksecret <CLAVE_PRECOMPARTIDA>     # debe ser idéntica a la de FGT-A
    next
end
```

### 3.4 Fase 2 IPsec (Phase2-interface)
```bash
config vpn ipsec phase2-interface
    edit "VPN-B-A-P2"
        set phase1name "VPN-B-A"
        set proposal des-sha256
        set dhgrp 14
        set src-subnet 192.168.99.0 255.255.255.0
        set dst-subnet 10.15.99.0 255.255.255.0
    next
end
```

### 3.5 Rutas estáticas
```bash
config router static
    edit 1
        set gateway 200.0.0.5
        set device "port1"
    next
    edit 2
        set dst 10.15.99.0 255.255.255.0
        set device "VPN-B-A"
    next
end
```

### 3.6 Políticas de firewall
```bash
config firewall policy
    edit 1
        set name "LAN-to-VPN"
        set srcintf "port2"
        set dstintf "VPN-B-A"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
    next
    edit 2
        set name "VPN-to-LAN"
        set srcintf "VPN-B-A"
        set dstintf "port2"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
    next
end
```

### Verificación
```bash
get system status
get system interface physical
show vpn ipsec phase1-interface
show vpn ipsec phase2-interface
diagnose vpn ike gateway list
diagnose vpn tunnel list
get vpn ipsec tunnel summary
get router info routing-table all
show firewall policy
```

---

## 4. VPC6 (Host LAN-A)

```bash
ip 10.15.99.2/24 10.15.99.1
```
*(comando equivalente en VPCS: `ip <dirección>/<máscara> <gateway>`)*

### Verificación
```bash
show
ping 192.168.99.2
trace 192.168.99.2
```

---

## 5. VPC7 (Host LAN-B)

```bash
ip 192.168.99.2/24 192.168.99.1
```

### Verificación
```bash
show
ping 10.15.99.2
trace 10.15.99.2
```

---

## 6. Comandos de diagnóstico adicionales (troubleshooting)

```bash
# Reiniciar la negociación IKE si el túnel queda en "connecting"
diagnose vpn ike gateway flush name VPN-A-B
diagnose vpn ike gateway flush name VPN-B-A

# Captura de paquetes en tiempo real sobre la interfaz del túnel
diagnose sniffer packet VPN-A-B 'icmp' 4
diagnose sniffer packet VPN-B-A 'icmp' 4

# Ver sesiones activas en el firewall
diagnose sys session list
```

---

## 7. Resultado esperado

| Verificación                              | Resultado esperado            |
|--------------------------------------------|----------------------------------|
| `diagnose vpn ike gateway list`            | `status: established`           |
| `get vpn ipsec tunnel summary`             | `selectors(total,up): 1/1`      |
| `diagnose vpn tunnel list`                 | `rxp`/`txp` > 0 después de ping |
| `get router info routing-table all`        | Ruta `S` hacia LAN remota vía túnel |
| `ping` desde VPC6 hacia VPC7               | Respuesta exitosa, `ttl=62`     |
| `ping` desde VPC7 hacia VPC6               | Respuesta exitosa, `ttl=62`     |
| `trace` (traceroute) en ambos sentidos      | Mínimo número de saltos, sin pasar visiblemente por la red pública |
