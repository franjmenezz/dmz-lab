# INFORME DE CONFIGURACIÓN DE DMZ CON CISCO PACKET TRACER

## 1. OBJETIVO DE APRENDIZAJE

El objetivo principal de este laboratorio era implementar y asegurar una Zona Desmilitarizada (DMZ) en un entorno de red corporativo simulado mediante Cisco Packet Tracer. Específicamente, se buscaba:

- **Configurar direccionamiento IP** estático en una topología compleja de tres segmentos de red (LAN interna, DMZ, red externa)
- **Implementar NAT estático** para exponer de forma segura un servidor web DMZ a Internet sin comprometer la red interna
- **Diseñar e implementar ACL extendidas** para controlar el tráfico de forma granular, permitiendo solo servicios web (HTTP/HTTPS) desde el exterior hacia la DMZ
- **Verificar seguridad y funcionalidad** mediante pruebas de conectividad y bloqueo selectivo de tráfico no deseado (ICMP, otros puertos)

---

## 2. TOPOLOGÍA DE RED

### 2.1 Descripción General

La topología implementada representa una infraestructura corporativa moderna con tres zonas de seguridad claramente delimitadas:

```
INTERNET (PC_External: 192.168.3.10)
         ↓ (ICMP bloqueado, TCP 80/443 permitido)
    [Router_FW] - NAT + ACL (Corazón de Seguridad)
    ↙          ↓          ↘
 (G0/0)      (G0/1)      (G0/2)
  192.168    192.168     192.168
  .3.1/.24   .1.1/.24    .2.1/.24
   ↓          ↓           ↓
[RED        [RED       [ZONA DMZ]
EXTERNA]   INTERNA]     
           PC_Internal  Server-PT
           192.168      Web_DMZ
           .1.10/.24    192.168.2.10
                        ↓ (HTTP en puerto 80)
```

### 2.2 Componentes de Red

| Dispositivo | Modelo/Sistema | Interfaz | IP Asignada | Máscara | Gateway | Función |
|---|---|---|---|---|---|---|
| **Router_FW** | Cisco ISR G2 4331 | G0/0 | 192.168.3.1 | 255.255.255.0 | — | Puerta externa, NAT |
| | | G0/1 | 192.168.1.1 | 255.255.255.0 | — | LAN interna |
| | | G0/2 | 192.168.2.1 | 255.255.255.0 | — | Red DMZ |
| **PC_External** | PC (Simulación) | FastEthernet0 | 192.168.3.10 | 255.255.255.0 | 192.168.3.1 | Cliente Internet |
| **PC_Internal** | PC (Simulación) | FastEthernet0 | 192.168.1.10 | 255.255.255.0 | 192.168.1.1 | Empleado LAN |
| **Server-PT Web_DMZ** | Servidor Web | FastEthernet0 | 192.168.2.10 | 255.255.255.0 | 192.168.2.1 | Servidor público |
| **Switch 2960** | Cisco 2960 | — | — | — | — | Segmentación LAN |

---

## 3. CONFIGURACIÓN IMPLEMENTADA

### 3.1 Paso 1: Configuración de Direccionamiento IP en Dispositivos Finales

Se asignaron direcciones IP estáticas en cada dispositivo siguiendo el esquema de subredes /24:

**PC_Internal (192.168.1.0/24 - LAN Roja):**
```
IP Address: 192.168.1.10
Subnet Mask: 255.255.255.0
Default Gateway: 192.168.1.1
DNS Server: 0.0.0.0 (no requerido para pruebas)
```

**Server-PT Web_DMZ (192.168.2.0/24 - Zona DMZ):**
```
IP Address: 192.168.2.10
Subnet Mask: 255.255.255.0
Default Gateway: 192.168.2.1
DNS Server: 0.0.0.0
```

**PC_External (192.168.3.0/24 - Red Externa/Internet):**
```
IP Address: 192.168.3.10
Subnet Mask: 255.255.255.0
Default Gateway: 192.168.3.1
DNS Server: 0.0.0.0
```

### 3.2 Paso 2: Configuración de Interfaces en Router_FW

Acceso a CLI del Router_FW y configuración de las tres interfaces Gigabit Ethernet:

```cisco
Router_FW# configure terminal
Router_FW(config)# hostname Router_FW

! Interfaz GigabitEthernet0/0 (Externa)
Router_FW(config)# interface GigabitEthernet0/0
Router_FW(config-if)# ip address 192.168.3.1 255.255.255.0
Router_FW(config-if)# no shutdown
Router_FW(config-if)# exit

! Interfaz GigabitEthernet0/1 (Interna)
Router_FW(config)# interface GigabitEthernet0/1
Router_FW(config-if)# ip address 192.168.1.1 255.255.255.0
Router_FW(config-if)# no shutdown
Router_FW(config-if)# exit

! Interfaz GigabitEthernet0/2 (DMZ)
Router_FW(config)# interface GigabitEthernet0/2
Router_FW(config-if)# ip address 192.168.2.1 255.255.255.0
Router_FW(config-if)# no shutdown
Router_FW(config-if)# exit

Router_FW(config)# end
Router_FW# write memory
```

**Estado:** ✅ Todas las interfaces activadas y con direccionamiento correcto.

### 3.3 Paso 3: Verificación de Conectividad Básica

Se realizaron pruebas de ping desde cada dispositivo hacia su respectivo gateway para verificar la capa 3:

| Origen | Destino | Comando | Resultado | RTT |
|---|---|---|---|---|
| PC_Internal | Router_FW G0/1 | ping 192.168.1.1 | 4/4 éxito (0% pérdida) | 3ms |
| Server-PT | Router_FW G0/2 | ping 192.168.2.1 | 4/4 éxito (0% pérdida) | 1ms |
| PC_External | Router_FW G0/0 | ping 192.168.3.1 | 4/4 éxito (0% pérdida) | 3ms |

**Conclusión:** Conectividad de red base verificada. Todos los hosts alcanzan sus gateways sin problemas.

### 3.4 Paso 4: Configuración de NAT Estático en Router_FW

Se implementó NAT estático "inside" en GigabitEthernet0/0 para traducir la dirección privada del servidor DMZ a la interfaz externa, permitiendo acceso desde Internet:

```cisco
Router_FW(config)# interface GigabitEthernet0/1
Router_FW(config-if)# ip nat inside
Router_FW(config-if)# exit

Router_FW(config)# interface GigabitEthernet0/2
Router_FW(config-if)# ip nat inside
Router_FW(config-if)# exit

Router_FW(config)# interface GigabitEthernet0/0
Router_FW(config-if)# ip nat outside
Router_FW(config-if)# exit

! Regla NAT: Mapeo estático del servidor DMZ a IP externa
Router_FW(config)# ip nat inside source static 192.168.2.10 192.168.3.1

Router_FW(config)# end
Router_FW# write memory
```

**Funcionamiento:**
- Tráfico externo dirigido a 192.168.3.1 es redirigido a 192.168.2.10 (servidor DMZ)
- Tráfico saliente desde DMZ es traducido a la IP 192.168.3.1 (invisible para Internet)

**Verificación:** `show ip nat translations` muestra mapeo activo.

### 3.5 Paso 5: Activación de Servicios Web en Servidor DMZ

Se habilitaron los servicios HTTP y HTTPS en el servidor web DMZ:

**Configuración de Services en Server-PT Web_DMZ:**
```
Desktop → Services
├─ HTTP: ✅ ON (Puerto 80)
├─ HTTPS: ✅ ON (Puerto 443)
└─ File Manager: 
   ├─ index.html (página de bienvenida)
   ├─ hello.html
   ├─ copyrights.html
   └─ Imágenes de soporte
```

**Validación:** El servidor responde correctamente a solicitudes HTTP desde cualquier origen que tenga conectividad IP hacia 192.168.2.10.

### 3.6 Paso 6: Prueba de Acceso Web Inicial (Antes de ACLs)

Se verificó que el acceso web funciona desde ambas PCs hacia el servidor DMZ:

**Desde PC_External:**
- URL: http://192.168.3.1 (redirigida vía NAT a 192.168.2.10)
- Resultado: ✅ Página de bienvenida Cisco Packet Tracer cargada
- Protocolo: TCP puerto 80 (HTTP)

**Desde PC_Internal:**
- URL: http://192.168.2.10 (acceso directo)
- Resultado: ✅ Página de bienvenida Cisco Packet Tracer cargada
- Protocolo: TCP puerto 80 (HTTP)

**Conclusión:** NAT funcional; todos pueden acceder a los servicios web del DMZ.

### 3.7 Paso 7: Configuración de ACL Extendida para Seguridad

Se implementó una ACL extendida (101) para controlar el tráfico de forma granular, aplicando el principio de "deny by default":

```cisco
Router_FW(config)# ip access-list extended 101

! Permitir TCP hacia puerto 80 (HTTP) desde cualquier origen
Router_FW(config-ext-nacl)# permit tcp any host 192.168.3.1 eq 80

! Permitir TCP hacia puerto 443 (HTTPS) desde cualquier origen
Router_FW(config-ext-nacl)# permit tcp any host 192.168.3.1 eq 443

! Permitir ICMP echo-reply hacia router (conexión establecida)
Router_FW(config-ext-nacl)# permit icmp any 192.168.3.0 0.0.0.255 echo-reply

! Permitir tráfico desde LAN interna a DMZ (puerto 80)
Router_FW(config-ext-nacl)# permit tcp 192.168.1.0 0.0.0.255 host 192.168.2.10 eq 80

! Permitir ICMP desde LAN interna a DMZ (echo-reply)
Router_FW(config-ext-nacl)# permit icmp any 192.168.1.0 0.0.0.255 echo-reply

! Denegar tráfico DMZ a LAN interna (seguridad crítica)
Router_FW(config-ext-nacl)# deny ip 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255

! Denegar cualquier otro tráfico
Router_FW(config-ext-nacl)# deny ip any any

Router_FW(config-ext-nacl)# end
```

**Aplicación en interfaces:**

```cisco
Router_FW(config)# interface GigabitEthernet0/0
Router_FW(config-if)# ip access-group 101 in
Router_FW(config-if)# exit

Router_FW(config)# interface GigabitEthernet0/1
Router_FW(config-if)# ip access-group 101 in
Router_FW(config-if)# exit

Router_FW(config)# end
Router_FW# write memory
```

### 3.8 Paso 8: Corrección de ACL para Bloquear ICMP desde Externa

**Problema Identificado:** PC_External podía hacer ping a 192.168.3.1, lo cual es un riesgo de reconocimiento. Se eliminó la regla permisiva de ICMP:

```cisco
Router_FW(config)# ip access-list extended 101
Router_FW(config-ext-nacl)# no 10
! Línea 10 contenía: permit icmp any any echo-reply
Router_FW(config-ext-nacl)# end
Router_FW# write memory
```

**Resultado:** Ping desde PC_External a 192.168.3.1 ahora falla (100% pérdida), mientras HTTP/HTTPS persisten funcionales.

---

## 4. RESULTADOS DE PRUEBAS

### 4.1 Tabla de Pruebas de Conectividad

| # | Origen | Destino | Protocolo | Puerto | Esperado | Resultado | Estado |
|---|---|---|---|---|---|---|---|
| 1 | PC_Internal | Router_FW(G0/1) | ICMP | — | Éxito | 4/4 exitosos (0% pérdida) | PASS |
| 2 | Server-PT DMZ | Router_FW(G0/2) | ICMP | — | Éxito | 4/4 exitosos (0% pérdida) | PASS |
| 3 | PC_External | Router_FW(G0/0) Web | TCP | 80 | Éxito | Página cargada (http://192.168.3.1) | PASS |
| 4 | PC_Internal | Server DMZ | TCP | 80 | Éxito | Página cargada (http://192.168.2.10) | PASS |
| 5 | Server-PT DMZ | PC_Internal | ICMP | — | Fallo (ACL) | 0/4 exitosos (100% pérdida) | PASS |
| 6 | PC_External | Router_FW(G0/0) Ping | ICMP | — | Fallo (ACL) | 0/4 exitosos (100% pérdida) | PASS |

### 4.2 Análisis de Resultados

**Accesos Permitidos:**
- HTTP (TCP:80) desde cualquier origen hacia servidor DMZ
- HTTPS (TCP:443) desde cualquier origen hacia servidor DMZ
- Conectividad IP dentro de cada subred (ICMP entre dispositivos y gateway)
- Tráfico de LAN interna hacia servicios web en DMZ

**Accesos Bloqueados (Intención de Seguridad):**
- ICMP (Ping) desde red externa hacia router/DMZ (reconocimiento bloqueado)
- Todo tráfico IP desde DMZ hacia LAN interna (prevención de movimiento lateral)
- Cualquier puerto/protocolo no explícitamente permitido

### 4.3 Auto-Evaluación en Packet Tracer

Se ejecutó "Check Results" en el laboratorio de Packet Tracer, obteniendo puntuación perfecta:

**Resultado Final: 100% - ACTIVIDAD COMPLETADA**

```
Congratulations Guest! You completed the activity.

Overall Feedback
Assessment Items | Connectivity Tests

Connectivity Tests Results:
┌─────────────────────────────────────────────────────────┐
│ # │ Status  │ Test Condition        │ Pts │ Type │
├─────────────────────────────────────────────────────────┤
│ 1 │ Correct │ PC_Int → Rtr_Int      │ 1   │ ICMP │
│ 2 │ Correct │ Server_DMZ → Rtr_DMZ  │ 1   │ ICMP │
│ 3 │ Correct │ PC_Ext → Rtr_Ext(Web) │ 1   │ TCP  │
│ 4 │ Correct │ PC_Int → Server_DMZ   │ 1   │ TCP  │
│ 5 │ Correct │ Server_DMZ → PC_Int   │ 2   │ ICMP │
│ 6 │ Correct │ PC_Ext → Rtr_Ext(Ping)│ 3   │ ICMP │
└─────────────────────────────────────────────────────────┘

Total Points: 9/9
Score: 100%
```

---

## 5. ANÁLISIS DE SEGURIDAD

### 5.1 Principios Aplicados

**1. Segmentación de Red:**
- Tres zonas claramente aisladas: LAN interna (confiable), DMZ (semi-confiable), externa (no confiable)
- Cada segmento en subred /24 diferente

**2. NAT (Network Address Translation):**
- Oculta infraestructura interna hacia Internet
- Servidor DMZ visible con IP pública traducida
- LAN interna completamente invisible (no hay mapeo NAT)

**3. Control de Acceso (ACL):**
- Whitelist: Solo HTTP/HTTPS permitidos explícitamente
- Deny by default: Cualquier tráfico no permitido es bloqueado
- Filtrado stateless: ACL en ambas direcciones

**4. Aislamiento DMZ-LAN:**
- Regla crítica: `deny ip 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255`
- Previene movimiento lateral si servidor DMZ es comprometido
- LAN interna solo puede iniciar conexiones a DMZ, no viceversa

### 5.2 Escenarios de Ataque Prevenidos

| Escenario | Ataque | Prevención |
|---|---|---|
| Reconocimiento | Ping ICMP desde Internet | ACL bloquea ICMP externo |
| Explotación | Acceso a puertos no HTTP/HTTPS | ACL whitelist (solo 80, 443) |
| Movimiento Lateral | Servidor comprometido pivota a LAN | Deny DMZ→LAN en ACL |
| IP Spoofing | Cliente externo se hace pasar por interno | NAT traduce solo salidas DMZ |
| Enumeración | Escaneo de puertos en router | No hay respuesta ICMP, puertos cerrados |

---

## 6. CONCLUSIONES

### 6.1 Objetivos Alcanzados

**Configuración IP:** Direccionamiento estático correcto en 3 segmentos de red  
**NAT Funcional:** Servidor DMZ (192.168.2.10) accesible desde Internet vía 192.168.3.1  
**ACL Implementada:** Control granular de tráfico basado en protocolo, puerto, origen y destino  
**Seguridad Validada:** Bloqueo de tráfico no deseado; aislamiento DMZ-LAN; reconocimiento prevenido  
**Pruebas Superadas:** 100% de test cases correctos en Packet Tracer  

### 6.2 Aprendizajes Clave

1. **NAT Estático:** Imprescindible para exponer servicios internos manteniendo privacidad
2. **ACL Extendidas:** Más potentes que estándar; permiten filtrado por protocolo, puerto, rango IP
3. **Deny by Default:** Filosofía de seguridad superior a "allow by default"
4. **Segmentación:** Fundamental para contener brechas de seguridad (defense-in-depth)
5. **Testing Iterativo:** Validación constante (ping, web) durante configuración

### 6.3 Recomendaciones para Producción

- **Reemplace ACL estáticas por Firewall Stateful (ASA, FTD)** para análisis de sesión
- **Implemente VPN** para comunicación segura entre redes internas
- **Agregue IDS/IPS** en DMZ para detección de intrusiones
- **Monitoreo continuo** de logs de acceso y tráfico de red
- **Actualización regular** de firmas de seguridad

---

## 7. ANEXOS

### 7.1 Comandos Clave Utilizados

**Configuración de Interfaces:**
```
Router_FW(config)# interface GigabitEthernet0/X
Router_FW(config-if)# ip address 192.168.X.1 255.255.255.0
Router_FW(config-if)# no shutdown
```

**Configuración NAT:**
```
Router_FW(config)# interface GigabitEthernet0/0
Router_FW(config-if)# ip nat outside
Router_FW(config)# ip nat inside source static 192.168.2.10 192.168.3.1
```

**Configuración ACL:**
```
Router_FW(config)# ip access-list extended 101
Router_FW(config-ext-nacl)# permit tcp any host 192.168.3.1 eq 80
Router_FW(config-ext-nacl)# interface GigabitEthernet0/0
Router_FW(config-if)# ip access-group 101 in
```

**Verificación:**
```
Router_FW# show ip nat translations
Router_FW# show access-lists
Router_FW# show interfaces brief
```

### 7.2 Rango de Subredes Implementadas

| Red | CIDR | Rango IP | Máscara | Gateway | Dispositivos |
|---|---|---|---|---|---|
| Interna (LAN) | 192.168.1.0/24 | .0–.255 | 255.255.255.0 | .1 | PC_Internal, Switch |
| DMZ | 192.168.2.0/24 | .0–.255 | 255.255.255.0 | .1 | Server-PT Web_DMZ |
| Externa | 192.168.3.0/24 | .0–.255 | 255.255.255.0 | .1 | PC_External, Internet |

### 7.3 Evidencia Fotográfica de Configuración

Se incluyen capturas de:
- Configuración de interfaces en CLI del router
- IP static en dispositivos finales
- Servicios HTTP activos en servidor DMZ
- Pruebas de ping exitosas
- Acceso web desde PC_External y PC_Internal
- Salida de "Check Results" con 100% de aprobación

---

## 8. REFLEXIÓN FINAL

Este laboratorio simuló de forma realista un escenario corporativo donde es crítico exponer servicios públicos (servidor web) sin comprometer la seguridad interna. La combinación de NAT + ACL + Segmentación demuestra cómo implementar "defense-in-depth" en redes profesionales. La validación del 100% de test cases confirma que la configuración es robusta, funcional y segura.

**Duración Total:** Aproximadamente 2 horas de configuración y pruebas  
**Herramienta:** Cisco Packet Tracer v8.x  
**Resultado:** PROYECTO COMPLETADO CON ÉXITO

---