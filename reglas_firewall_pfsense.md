# Reglas de seguridad del laboratorio

Este documento explica qué permite y qué bloquea el "portero" del laboratorio (el firewall pfSense), por qué cada permiso existe, y por qué puerta de cada equipo entra o sale la información.

**Última actualización:** 10 de octubre de 2026

---

## Cómo funciona

El laboratorio está dividido en cuatro zonas, cada una con su propia red. Los equipos de una zona no pueden hablar con los de otra salvo que exista un permiso explícito. El firewall revisa cada petición y decide si la deja pasar.

Hay algunas reglas básicas:

- **Todo lo que no está permitido está bloqueado.** No hace falta añadir reglas de bloqueo.
- **Cada permiso tiene un código** (por ejemplo, SOC-05). Si algo falla, ese código permite buscar en el registro del firewall qué regla está actuando.
- **"Permitido" no significa "sin control".** Algunos permisos son amplios a propósito, para que el laboratorio funcione. Se indican en la sección de pendientes.

### Glosario rápido

- **Puerto:** cada equipo tiene muchas "puertas" numeradas. Cada servicio usa una puerta concreta (por ejemplo, la web usa la 443). Un permiso siempre indica a qué puerta se puede llamar.
- **TCP y UDP:** dos formas de enviar información. TCP es ordenado y fiable (como una carta certificada). UDP es más rápido y sencillo (como un aviso sin acuse de recibo).
- **ICMP:** el "ping", que sirve para comprobar si un equipo responde.

---

## Las zonas del laboratorio

| Zona | Dirección de red | Qué hay dentro | Para qué sirve |
|---|---|---|---|
| **Corporate** | 192.168.10.0/24 | Controlador de dominio DC01 (192.168.10.10) y equipo de trabajo WIN01 (192.168.10.20) | Simula una empresa con sus usuarios y equipos |
| **Red Team** | 192.168.30.0/24 | Nada por ahora | Reservada para pruebas ofensivas aisladas |
| **SOC** | 192.168.20.0/24 | Wazuh WAZUH01 (192.168.20.10), Elastic ELK01 (192.168.20.11) y puesto del analista ANALYST01 (192.168.20.20) | Centro donde se vigila y analiza lo que ocurre en la empresa |
| **Externa (Kali)** | 192.168.40.0/24 | Equipo Kali Linux RT01 (192.168.40.10) | Equipo de pruebas que simula un atacante desde fuera |

El firewall pfSense tiene una dirección en cada zona (la "puerta de salida" de cada red): 192.168.10.1, 192.168.30.1, 192.168.20.1 y 192.168.40.1.

---

## Permisos de la zona Corporate

Los equipos de la empresa pueden enviar información de vigilancia al centro SOC.

| Código | Qué permite | Protocolo | Desde | Hacia | Puerto | Para qué |
|---|---|---|---|---|---|---|
| LAN-01 | Registro y envío de eventos de los agentes Wazuh | TCP | Corporate | WAZUH01 (192.168.20.10) | 1514 a 1515 | 1515 sirve para darse de alta; 1514 para enviar los eventos de seguridad |
| LAN-02 | Envío de eventos de los agentes por otro canal | UDP | Corporate | WAZUH01 (192.168.20.10) | 1514 | Complementa al anterior para no perder información |
| LAN-03 | Envío de datos a Logstash | TCP | Corporate | ELK01 (192.168.20.11) | 5044 | **No se utiliza actualmente.** Puede eliminarse |
| LAN-04 | Acceso libre a Internet y a la zona SOC | Cualquiera | Corporate | Cualquiera | Cualquiera | Navegar, actualizar Windows y que los equipos funcionen con normalidad. Se revisará más adelante |

---

## Permisos de la zona Red Team

No hay permisos. La zona queda aislada del resto.

---

## Permisos de la zona SOC

Esta zona tiene varios tipos de permisos, según su finalidad.

### Administrar el firewall y salir a Internet

| Código | Qué permite | Protocolo | Desde | Hacia | Puerto | Para qué |
|---|---|---|---|---|---|---|
| SOC-01 | Comprobar que el firewall responde | ICMP (ping) | Zona SOC | Firewall (192.168.20.1) | No aplica | Diagnosticar si la red funciona |
| SOC-02 | Entrar a la página de configuración del firewall | TCP | Zona SOC | Firewall (192.168.20.1) | 443 | Administrar el firewall desde la zona SOC |
| SOC-03 | Preguntar nombres de Internet al firewall | TCP | Zona SOC | Firewall (192.168.20.1) | 53 | Que los servidores encuentren otros sitios por su nombre |
| SOC-04 | Igual que el anterior, por otro canal | UDP | Zona SOC | Firewall (192.168.20.1) | 53 | Complementa al anterior |
| SOC-05 | Descargar actualizaciones y programas | TCP | Zona SOC | Cualquiera | 80 y 443 | Los servidores necesitan actualizarse y descargar sus componentes |

### Unir el puesto del analista al dominio de la empresa

Estos permisos van dirigidos únicamente al controlador de dominio DC01 (192.168.10.10), desde el puesto ANALYST01 (192.168.20.20). Son necesarios para que el puesto forme parte de la empresa y use sus cuentas de usuario.

| Código | Qué permite | Protocolo | Puerto | Para qué |
|---|---|---|---|---|
| SOC-06 | Preguntar por los nombres de la red de la empresa | TCP | 53 | Que ANALYST01 encuentre el dominio |
| SOC-07 | Igual que el anterior, por otro canal | UDP | 53 | Complementa al anterior |
| SOC-08 | Verificar la identidad del usuario al entrar | TCP | 88 | Que el usuario entre con su cuenta de la empresa (Kerberos) |
| SOC-09 | Igual que el anterior, por otro canal | UDP | 88 | Complementa al anterior |
| SOC-10 | Consultar el directorio de usuarios | TCP | 389 | Saber quién es quién en la empresa (LDAP) |
| SOC-11 | Igual que el anterior, por otro canal | UDP | 389 | Complementa al anterior |
| SOC-12 | Acceder a carpetas compartidas y normas de la empresa | TCP | 445 | Aplicar las políticas de configuración de la empresa (SMB) |
| SOC-13 | Localizar servicios de la empresa | TCP | 135 | Requisito para que Windows hable con el controlador (RPC) |
| SOC-14 | Consultar el directorio con conexión cifrada | TCP | 636 | Mayor seguridad en el acceso a usuarios (LDAPS) |
| SOC-15 | Buscar usuarios en toda la empresa | TCP | 3268 | Localizar cuentas aunque estén en otra parte del directorio |
| SOC-16 | Sincronizar la hora con el controlador de dominio | UDP | 123 | Una hora desajustada impide el inicio de sesión (NTP) |
| SOC-17 | Abrir conexiones temporales de Windows | TCP | 49152 a 65535 | Necesario para completar la unión al dominio. **Es un permiso amplio** que se ajustará cuando el laboratorio esté estable |

### Permisos opcionales (pendientes)

| Código | Qué permitiría | Protocolo | Desde | Hacia | Puerto | Cuándo hace falta |
|---|---|---|---|---|---|---|
| SOC-18 | Cambiar la contraseña desde ANALYST01 | TCP | ANALYST01 | DC01 | 464 | Si se quiere cambiar contraseñas desde ese equipo |
| SOC-19 | Igual que el anterior, por otro canal | UDP | ANALYST01 | DC01 | 464 | Complementa al anterior |
| SOC-20 | Sincronizar la hora de los servidores | UDP | Zona SOC | DC01 | 123 | Si se quiere que Wazuh y Elastic tengan la misma hora de la empresa |

### Comunicaciones dentro de la zona SOC

Estas comunicaciones no necesitan permiso porque los equipos están en la misma zona:

| Desde | Hacia | Puerto | Para qué |
|---|---|---|---|
| WAZUH01 (192.168.20.10) | ELK01 (192.168.20.11) | 9200 | Envío de las alertas de seguridad a Elastic |
| ANALYST01 (192.168.20.20) | WAZUH01 (192.168.20.10) | 443 | Ver el panel de Wazuh |
| ANALYST01 (192.168.20.20) | ELK01 (192.168.20.11) | 5601 | Ver Kibana, el panel de búsqueda de Elastic |

---

## Permisos de la zona Externa (Kali)

| Código | Qué permite | Protocolo | Desde | Hacia | Puerto | Para qué |
|---|---|---|---|---|---|---|
| OPT3-01 | Atacar la zona Corporate | Cualquiera | Kali (192.168.40.0/24) | Corporate (192.168.10.0/24) | Cualquiera | Es el objetivo del laboratorio: simular un atacante. **Permite todo** y puede limitarse a DC01 y WIN01 |
| OPT3-02 | Descargar herramientas por web | TCP | Kali | Cualquiera | 80 | Instalar programas de prueba (HTTP) |
| OPT3-03 | Descargar herramientas por web segura | TCP | Kali | Cualquiera | 443 | Igual que el anterior, con cifrado (HTTPS) |

El equipo de pruebas **no tiene acceso** a la zona SOC. Así, el atacante simulado no puede tocar los servidores de vigilancia.

---

## Pendientes y recomendaciones

1. **Quitar LAN-03**, porque no se usa.
2. **Revisar LAN-04** (acceso libre desde Corporate) cuando LAN-01 y LAN-02 estén comprobados.
3. **Ajustar SOC-17** cuando el dominio esté estable, para dejar solo los puertos necesarios.
4. **Limitar OPT3-01** a los equipos de la empresa que se quieren probar (DC01 y WIN01).
5. **Cambiar la contraseña del usuario de administración de Elastic**, porque se mostró en una conversación.

---

## Anexo técnico (para el equipo de TI)

Comprobaciones de cada permiso, desde el equipo indicado:

| Código | Comando de prueba |
|---|---|
| LAN-01 | `Test-NetConnection 192.168.20.10 -Port 1515` (desde WIN01) |
| SOC-01 | `ping 192.168.20.1` (desde ANALYST01) |
| SOC-02 | `Test-NetConnection 192.168.20.1 -Port 443` |
| SOC-03 y SOC-04 | `nslookup google.com 192.168.20.1` |
| SOC-05 | `curl -I https://archive.ubuntu.com` (desde WAZUH01) |
| SOC-06 y SOC-07 | `Resolve-DnsName lab.local` |
| SOC-08 | `Test-NetConnection 192.168.10.10 -Port 88` |
| SOC-10 | `Test-NetConnection 192.168.10.10 -Port 389` |
| SOC-12 | `Test-NetConnection 192.168.10.10 -Port 445` |
| SOC-13 | `Test-NetConnection 192.168.10.10 -Port 135` |
| SOC-16 | `w32tm /stripchart /computer:192.168.10.10 /samples:3 /dataonly` |
| SOC-17 | `Test-ComputerSecureChannel` (en ANALYST01) |

---

## Historial de cambios

| Fecha | Cambio |
|---|---|
| 10/10/2026 | Primera versión del documento: zonas, permisos, puertos y pendientes. |
