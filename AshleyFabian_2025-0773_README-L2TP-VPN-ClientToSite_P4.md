# Práctica 4 — VPN Client-to-Site: IPSec IKEv1 con L2TP

**Estudiante:** Ashley Fabian
**Matrícula:** 2025-0773
**Institución:** Instituto Tecnológico Las Américas (ITLA)
**Herramienta:** GNS3 (CSR1000v, IOSvL2, VPCS) + VMware + Windows Client
**Direccionamiento base:** 25.7.73.0/24

---

## 1. Introducción

Esta documentación describe la configuración de una **VPN Client-to-Site
punto a multipunto**, utilizando **IPSec IKEv1 combinado con L2TP**, para
permitir que un cliente Windows externo se conecte de forma segura a una
LAN protegida a través de un túnel cifrado.

Esta es la novena y última topología de la Práctica 4, complementando las
seis VPN site-to-site (IKEv1/IKEv2 en modalidad basada en políticas, VTI y
GRE sobre IPSec) y las dos topologías DMVPN (Fase 2 y Fase 3).

---

## 2. Topología

```
[LAN-Host] --- [SW-LAN (IOSvL2)] --- [R-LAN (CSR1000v)] --- [R-ISP] --- [Cloud/NAT] --- [Windows Client VM]
                                    (VPN Server L2TP/IPSec)              (Internet simulado)
```

> 📸 **CAPTURA 1:** Vista completa de la topología en el workspace de
> GNS3, con todos los nodos visibles y las conexiones etiquetadas.

![](./Topologia.png)

### 2.1 Direccionamiento

| Segmento | Red | Notas |
|---|---|---|
| WAN R-LAN ↔ R-ISP | 25.7.73.0/30 | R-LAN: .2, R-ISP: .1 |
| WAN R-ISP ↔ Cliente | 25.7.73.4/30 | R-ISP: .5, Cliente: .6 |
| LAN interna (protegida) | 25.7.73.32/27 | R-LAN: .33 (gateway), Host: .34 |
| Pool L2TP (IP virtual al cliente) | 25.7.73.64/28 | Asignado dinámicamente (.66–.78) |

---

## 3. Configuración de los equipos

### 3.1 R-ISP

```
enable
configure terminal
hostname R-ISP

interface G1
 ip address 25.7.73.1 255.255.255.252
 no shutdown
 exit

interface G2
 ip address 25.7.73.5 255.255.255.252
 no shutdown
 exit

ip route 25.7.73.32 255.255.255.224 25.7.73.2

end
write memory
```

> 📸 **CAPTURA 2:** `show running-config` de R-ISP mostrando las
> interfaces G1 y G2 con sus IPs, y la ruta estática configurada.

![](./RunISP.png)

---

### 3.2 R-LAN (CSR1000v) — Servidor VPN L2TP/IPSec IKEv1

```
enable
configure terminal
hostname R-LAN

interface G1
 ip address 25.7.73.2 255.255.255.252
 no shutdown
 exit

interface G2
 ip address 25.7.73.33 255.255.255.224
 no shutdown
 exit

ip route 0.0.0.0 0.0.0.0 25.7.73.1

! AAA local para autenticación PPP
aaa new-model
aaa authentication ppp L2TP-AUTH local
aaa authorization network L2TP-AUTHOR local

username vpnuser password 0 Vpn2025Fabian

! Política IKE Fase 1 (IKEv1)
! AES-256 / SHA / DH Group 14: coincide con la propuesta por defecto
! del cliente VPN nativo de Windows
crypto isakmp policy 5
 encr aes 256
 hash sha
 authentication pre-share
 group 14
 lifetime 86400
 exit

crypto isakmp key Preshared2025Key address 25.7.73.6

! Transform-set IPSec en modo transporte (obligatorio para L2TP)
crypto ipsec transform-set L2TP-SET esp-aes esp-sha-hmac
 mode transport
 exit

! ACL de tráfico interesante: UDP 1701 en origen Y destino,
! que es como el cliente L2TP nativo de Windows negocia el proxy-ID
access-list 100 permit udp any eq 1701 any eq 1701

! Crypto map dinámico (el peer no tiene IP fija garantizada)
crypto dynamic-map L2TP-DYNMAP 10
 set transform-set L2TP-SET
 match address 100
 exit

crypto map L2TP-MAP 10 ipsec-isakmp dynamic L2TP-DYNMAP

interface G1
 crypto map L2TP-MAP
 exit

! VPDN (servidor L2TP)
vpdn enable

vpdn-group L2TP-GROUP
 accept-dialin
  protocol l2tp
  virtual-template 1
 exit
 no l2tp tunnel authentication
 exit

! Virtual-Template: asigna IP al cliente y autentica PPP
ip local pool L2TP-POOL 25.7.73.66 25.7.73.78

interface Virtual-Template1
 ip unnumbered G2
 peer default ip address pool L2TP-POOL
 ppp authentication ms-chap-v2 L2TP-AUTH
 exit

end
write memory
```

> 📸 **CAPTURA 3:** `show running-config` completo de R-LAN, visible
> desde `crypto isakmp policy 5` hasta `interface Virtual-Template1`.

![](./RunRLAN1.png)
![](./RunRLAN2.png)
![](./RunRLAN3.png)
![](./RunRLAN4.png)
![](./RunRLAN5.png)

---

### 3.3 SW-LAN (IOSvL2)

```
enable
configure terminal
hostname SW-LAN

vlan 10
 name LAN_PROTEGIDA
 exit

interface range Gi0/0 - 1
 switchport mode access
 switchport access vlan 10
 no shutdown
 exit

end
write memory
```

> 📸 **CAPTURA 4:** `show vlan brief` en SW-LAN mostrando la VLAN 10 con
> los puertos Gi0/0 y Gi0/1 asignados.

![](./VlanbriefSWITCH.png)

---

### 3.4 LAN-Host (VPCS)

```
ip 25.7.73.34 255.255.255.224 25.7.73.33
save
```

> 📸 **CAPTURA 5:** Salida de `show ip` en LAN-Host confirmando IP y
> gateway.

![](./SHIPPC.png)
---

### 3.5 Windows Client VM — Adaptador de red (segmento público)

| Parámetro | Valor |
|---|---|
| IP | 25.7.73.6 |
| Máscara | 255.255.255.252 |
| Puerta de enlace | 25.7.73.5 |

> 📸 **CAPTURA 6:** Ventana de propiedades de "Protocolo de Internet
> versión 4 (TCP/IPv4)" con estos valores configurados.

![](./ipWindows.png)
---

### 3.6 Windows Client VM — Conexión VPN

*Configuración → Red e Internet → VPN → Agregar una conexión VPN*

| Parámetro | Valor |
|---|---|
| Proveedor de VPN | Windows (integrado) |
| Nombre de conexión | P4-L2TP-VPN |
| Servidor | 25.7.73.2 |
| Tipo de VPN | L2TP/IPSec con clave precompartida |
| Clave precompartida | Preshared2025Key |
| Usuario | vpnuser |
| Contraseña | Vpn2025Fabian |

> 📸 **CAPTURA 7:** Pantalla de configuración de la conexión VPN en
> Windows con todos los campos anteriores visibles.

![](./VPNenWidnoes.png)

**Ajuste de registro** (requerido porque el servidor VPN no es Windows):

```
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\PolicyAgent
  Nombre: AssumeUDPEncapsulationContextOnSendRule (DWORD 32 bits)
  Valor:  2
```

> 📸 **CAPTURA 8:** Ventana de `regedit` mostrando la clave
> `AssumeUDPEncapsulationContextOnSendRule` creada con valor `2`.

![](assume.png)
---

## 4. Verificación de conectividad básica (pre-VPN)

Antes de establecer el túnel, se confirmó conectividad IP de extremo a
extremo sin cifrado:

```
R-LAN# ping 25.7.73.1
PC1> ping 25.7.73.33
```

> 📸 **CAPTURA 9:** Ping exitoso de R-LAN hacia R-ISP (25.7.73.1).

![](RLANping.png)
> 📸 **CAPTURA 10:** Ping exitoso de LAN-Host hacia su gateway
> (25.7.73.33).

![](PC1ping.png)

Desde Windows:

```
ping 25.7.73.5
ping 25.7.73.2
```

> 📸 **CAPTURA 11:** Ambos pings exitosos desde la terminal (cmd) de
> Windows.

![](Windowsping.png)

---

## 5. Establecimiento del túnel VPN

Con `debug crypto isakmp` activo en R-LAN, se inició la conexión desde
Windows. La negociación pasó por:

1. **Fase 1 (ISAKMP):** autenticación exitosa con `SA has been
   authenticated with 25.7.73.6`.
2. **Fase 2 (IPSec Quick Mode):** SA instalada con el transform-set
   `esp-aes esp-sha-hmac` en modo transporte.
3. **Subida de la interfaz Virtual-Access:**
   `%LINEPROTO-5-UPDOWN: Line protocol on Interface Virtual-Access2,
   changed state to up`.

> 📸 **CAPTURA 12:** Notificación/ícono de red en Windows mostrando el
> estado **"Conectado"** de `P4-L2TP-VPN`.
![](image-1.png)

---

## 6. Verificación final

**En R-LAN:**

```
show vpdn session
show users
show ip interface brief | include Virtual-Access
```

> 📸 **CAPTURA 13:** Salida de `show vpdn session` mostrando la sesión
> activa de `vpnuser`.

![](vpdnRLAN.png)

> 📸 **CAPTURA 14:** Salida de `show users` mostrando la interfaz
> `Vi2.1`, usuario `vpnuser` e IP `25.7.73.66`.

![](shusersRLAN.png)

> 📸 **CAPTURA 15:** Salida de `show ip interface brief | include
> Virtual-Access` con Virtual-Access2 en estado up/up.

![](briefraln.png)
**En Windows:**

```
ipconfig
ping 25.7.73.34
```

> 📸 **CAPTURA 16:** Salida de `ipconfig` mostrando la IP virtual
> recibida (25.7.73.66).

![](ipconfigWindows.png)

> 📸 **CAPTURA 17:** Ping exitoso desde Windows hacia `25.7.73.34`
> (LAN-Host), confirmando acceso completo a la red protegida a través
> del túnel.

![](Pingapc1dW.png)
---

## 7. Conclusión

Se estableció exitosamente una VPN Client-to-Site basada en IPSec IKEv1
con L2TP, permitiendo que un cliente Windows externo accediera a la LAN
protegida detrás de R-LAN. La configuración incluyó autenticación AAA
local, una política IKE alineada con las propuestas del cliente nativo de
Windows, un transform-set en modo transporte, y un pool de direcciones
asignado dinámicamente vía PPP/L2TP. Con esta topología se completan las
nueve VPN requeridas en la Práctica 4.

