# 📡 MikroTik RouterOS v7 - Configuración básica + WireGuard VPN (Puerto 51820)

Este documento contiene los pasos para configurar un router MikroTik desde cero:

- Conexión a Internet (por ether1)
- Red local LAN (por ether2)
- Servidor VPN usando **WireGuard** (puerto 51820)

---

## ✅ Requisitos

- MikroTik con **RouterOS v7.x**
- Acceso por Winbox o terminal
- Puerto `ether1` conectado al proveedor de Internet (WAN)
- Puerto `ether2` conectado a tu red LAN
- IP pública (fija o dinámica con DDNS)
- Cliente WireGuard (Windows, Android, iOS, Linux, etc.)

---

## 🛠️ Parte 1: Configuración básica de red

### 1. Restablecer configuración (opcional)

```
/system reset-configuration no-defaults=yes skip-backup=yes
```
⚠️ Esto borra toda la configuración previa. Úsalo solo si estás comenzando desde cero.

### 2. Configurar WAN (ether1)
```
1. Asigna DHCP a la interfaz WAN:
/interface ethernet
set [find default-name=ether1] name=WAN

/ip dhcp-client
add interface=WAN use-peer-dns=yes use-peer-ntp=yes disabled=no

2. Asigna IP estática a la interfaz WAN:

/ip address
add address=192.168.1.200/24 interface=WAN network=192.168.1.0

⚠️ Reemplaza /24 con la máscara correcta si tu red es diferente. /24 = 255.255.255.0

3. Configura el gateway (puerta de enlace) de tu red (por ejemplo, si el router del proveedor es 192.168.1.1):

/ip route
add dst-address=0.0.0.0/0 gateway=192.168.1.1
4. (Opcional) Configura servidores DNS manualmente:

/ip dns
set servers=1.1.1.1,8.8.8.8 allow-remote-requests=yes

❌ Elimina el cliente DHCP en WAN si lo tenías:

/ip dhcp-client
remove [find interface=WAN]
```
### 3. Configurar LAN (ether2)
```
/interface ethernet
set [find default-name=ether2] name=LAN

/ip address
add address=192.168.88.1/24 interface=LAN network=192.168.88.0

/ip pool
add name=dhcp_pool ranges=192.168.88.10-192.168.88.254

/ip dhcp-server
add address-pool=dhcp_pool interface=LAN name=dhcp1

/ip dhcp-server network
add address=192.168.88.0/24 gateway=192.168.88.1 dns-server=1.1.1.1,8.8.8.8

/ip dhcp-server enable dhcp1
```
### 4. Habilitar NAT para salida a Internet
```
/ip firewall nat
add chain=srcnat out-interface=WAN action=masquerade
```
### 5. Configurar DNS
```
/ip dns
set servers=1.1.1.1,8.8.8.8 allow-remote-requests=yes

🔐 Parte 2: Servidor VPN con WireGuard
```

### 6. Crear interfaz WireGuard (puerto 51820)
```
/interface/wireguard
add name=wg0 listen-port=51820

La clave privada se genera automáticamente. Puedes verla con:

/interface/wireguard/print detail
```
###  7. Asignar IP al túnel VPN
```
/ip address
add address=10.10.10.1/24 interface=wg0
```
### 8. Permitir tráfico VPN en el firewall
```
/ip firewall filter
add chain=input action=accept protocol=udp dst-port=51820 comment="Permitir WireGuard VPN"
add chain=input src-address=10.10.10.0/24 action=accept comment="Permitir acceso VPN al Router"
add chain=forward src-address=10.10.10.0/24 action=accept comment="Permitir tráfico desde VPN"
```
### 9. NAT para los clientes VPN
```
/ip firewall nat
add chain=srcnat src-address=10.10.10.0/24 out-interface=WAN action=masquerade
```
### 10. Agregar cliente VPN (peer)
Desde tu cliente (PC o móvil), genera claves públicas/privadas y luego:
```
/interface/wireguard/peers
add interface=wg0 public-key="CLAVE_PUBLICA_DEL_CLIENTE" allowed-address=10.10.10.2/32
```
```
💻 Configuración del Cliente WireGuard (ejemplo)

[Interface]
PrivateKey = CLIENT_PRIVATE_KEY
Address = 10.10.10.2/32
DNS = 1.1.1.1

[Peer]
PublicKey = PUBLIC_KEY_DEL_ROUTER
Endpoint = TU_IP_PUBLICA_O_DDNS:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
🧪 Verificación
Ver interfaz y claves:

/interface/wireguard/print detail
Ver estadísticas de conexión:

/interface/wireguard/peers/print stats
🌐 IP dinámica: activar DDNS

/ip cloud set ddns-enabled=yes
Tu dominio será algo como: xxxxx.sn.mynetname.net

Úsalo como Endpoint en la configuración del cliente.

➕ Agregar más clientes VPN
Solo repite el paso 10 con una IP diferente, como 10.10.10.3/32, 10.10.10.4/32, etc.
```

### Si quieres usar una DDNS (Dynamic DNS) con tu MikroTik para acceder a él remotamente incluso si tu IP cambia, puedes hacerlo de forma muy sencilla con el servicio gratuito que incluye MikroTik: IP Cloud, o usando otros servicios externos como DuckDNS, No-IP, etc.

✅ Opción recomendada: Usar MikroTik Cloud DDNS
MikroTik ya incluye un servicio DDNS gratuito, y es el más fácil de usar.

🔧 Activarlo:
```
/ip cloud set ddns-enabled=yes update-time=yes
🔍 Ver tu dominio DDNS:

/ip cloud print
Te mostrará algo así:

ddns-enabled: yes
dns-name: xxxxxx.sn.mynetname.net
public-address: xx.xx.xx.xx
Ese dominio xxxxxx.sn.mynetname.net será tu DDNS, que puedes usar como Endpoint en tu cliente WireGuard o para acceder a tu red remotamente.
```
