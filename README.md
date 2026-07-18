# 🖥️ RDP RemoteApp + IIS + NPS (RADIUS) + AAA en Router

<div align="center">

**Instituto Tecnológico de las Américas — ITLA**  
**Arlene Fernández Herrera · Matrícula: 2025-0730**

![Windows](https://img.shields.io/badge/SO-Windows%20Server%202016+-blue?style=for-the-badge&logo=windows)
![RDP](https://img.shields.io/badge/Servicio-RDP%20RemoteApp-green?style=for-the-badge)
![NPS](https://img.shields.io/badge/Servicio-NPS%20RADIUS-orange?style=for-the-badge)
![IIS](https://img.shields.io/badge/Servicio-IIS-lightblue?style=for-the-badge)

</div>

---

## 📑 Tabla de Contenidos

1. [Objetivo](#1-objetivo)
2. [Topología](#2-topología)
3. [Direccionamiento IP](#3-direccionamiento-ip)
4. [Configuración de RDP RemoteApp](#4-configuración-de-rdp-remoteapp)
5. [Configuración de RDP RemoteApp Web Client](#5-configuración-de-rdp-remoteapp-web-client)
6. [Configuración de IIS y Página Personalizada](#6-configuración-de-iis-y-página-personalizada)
7. [Publicar IIS en RDP RemoteApp](#7-publicar-iis-en-rdp-remoteapp)
8. [Configuración de NPS (RADIUS Server)](#8-configuración-de-nps-radius-server)
9. [Configuración del Router — AAA con RADIUS](#9-configuración-del-router--aaa-con-radius)
10. [Pruebas desde el Cliente](#10-pruebas-desde-el-cliente)
11. [Capturas de Pantalla](#11-capturas-de-pantalla)
12. [Video Demostrativo](#12-video-demostrativo)

---

## 1. Objetivo

Implementar y verificar una infraestructura de acceso remoto y autenticación centralizada que integra los siguientes servicios en **Windows Server 2016+**:

- **RDP RemoteApp**: publicar aplicaciones remotas accesibles sin escritorio completo.
- **RDP RemoteApp Web Client**: acceso a RemoteApp desde navegador web sin instalar cliente RDP.
- **IIS**: servidor web con página personalizada publicada como RemoteApp.
- **NPS (Network Policy Server)**: servidor RADIUS que autentica usuarios con niveles de acceso diferenciados (nivel 15 y nivel 1).
- **Router Cisco con AAA**: autenticación SSH vía RADIUS con logs de debug y autorización por niveles.

---

## 2. Topología

```
                           INTERNET
                               │
                               │
                       ┌───────┴───────┐
                       │   Router R1   │
                       │  (AAA+RADIUS) │
                       └───────┬───────┘
                               │
                               │
                       ┌───────┴───────┐
                       │   Switch SW1  │
                       └───────┬───────┘
                               │
               ┌───────────────┴────────────────┐
               │                                │
       ┌───────┴────────┐              ┌────────┴────────┐
       │   VPC1         │              │    Server1      │
       │  (Cliente)     │              │ Windows Server  │
       │  VM o Host     │              │  • RDS/RemoteApp│
       └────────────────┘              │  • IIS          │
                                       │  • NPS (RADIUS) │
                                       └─────────────────┘
```

### Dispositivos

| Dispositivo | Rol | Servicios |
|---|---|---|
| **R1** | Router Cisco IOS | AAA con RADIUS, SSH, usuario local de respaldo |
| **SW1** | Switch capa 2 | Interconexión LAN |
| **VPC1** | Cliente (VM o máquina Host) | Acceso RDP RemoteApp Web + SSH al router |
| **Server1** | Windows Server 2016+ | RDS RemoteApp, IIS, NPS (RADIUS Server) |

---

## 3. Direccionamiento IP

| Dispositivo  | Interfaz   | Dirección IP     | Máscara | Gateway      | Rol                          |
|--------------|------------|------------------|---------|--------------|------------------------------|
| **R1**       | (WAN)      | (según entorno)  | /24     | ISP          | Router — AAA + RADIUS        |
| **R1**       | (LAN)      | (según entorno)  | /24     | —            | Gateway LAN                  |
| **Server1**  | NIC        | (según entorno)  | /24     | IP R1 LAN    | RDS + IIS + NPS (RADIUS)     |
| **VPC1**     | NIC / eth0 | (según entorno)  | /24     | IP R1 LAN    | Cliente RDP Web + SSH        |

> Completar las IPs según el direccionamiento asignado en el entorno de laboratorio.

---

## 4. Configuración de RDP RemoteApp

> Toda la configuración se realiza mediante GUI — **Server Manager** y **Remote Desktop Services**.

### 4.1 Instalar el rol Remote Desktop Services

📸 *Ver captura: [evidencias/01_roles_instalados.png](#11-capturas-de-pantalla)*

1. Abrir **Server Manager** → `Manage` → `Add Roles and Features`.
2. Seleccionar **Remote Desktop Services installation**.
3. Tipo de instalación: **Quick Start** (para lab) o **Standard deployment**.
4. Seleccionar los servicios de rol:
   - ✅ **RD Session Host** — ejecuta las aplicaciones RemoteApp.
   - ✅ **RD Web Access** — portal web para acceder a RemoteApp.
   - ✅ **RD Connection Broker** — gestiona las conexiones.
5. Completar la instalación y **reiniciar** el servidor cuando se solicite.

---

### 4.2 Crear la colección RemoteApp

📸 *Ver captura: [evidencias/02_remoteapp_coleccion.png](#11-capturas-de-pantalla)*

1. En **Server Manager** → `Remote Desktop Services` → `Collections`.
2. Clic en `Tasks` → **Create Session Collection**.
3. Asignar nombre a la colección: `RemoteApp-ITLA`.
4. Seleccionar el servidor RD Session Host (el mismo Windows Server).
5. Agregar grupos de usuarios que tendrán acceso: `Domain Users` o un grupo específico.
6. Finalizar el asistente.

---

### 4.3 Publicar una aplicación como RemoteApp

📸 *Ver captura: [evidencias/02_remoteapp_coleccion.png](#11-capturas-de-pantalla)*

1. Dentro de la colección `RemoteApp-ITLA` → clic en `Tasks` → **Publish RemoteApp Programs**.
2. En la lista de programas, seleccionar la aplicación a publicar (ej. Internet Explorer / Edge para abrir la página IIS).
3. Clic en **Next** → **Publish** → **Close**.
4. La aplicación aparece en la colección lista para ser accedida.

---

## 5. Configuración de RDP RemoteApp Web Client

> El Web Client permite acceder a RemoteApp desde el **navegador** sin instalar el cliente RDP clásico.

### 5.1 Instalar el módulo RD Web Client

📸 *Ver captura: [evidencias/03_webclient_iis.png](#11-capturas-de-pantalla)*

Abrir **PowerShell como Administrador** en el Windows Server y ejecutar:

```powershell
# Instalar el módulo de gestión del Web Client
Install-Module -Name RDWebClientManagement -Force

# Importar el módulo
Import-Module RDWebClientManagement

# Instalar el Web Client (descarga la última versión)
Install-RDWebClientPackage
```

---

### 5.2 Configurar el certificado para el Web Client

📸 *Ver captura: [evidencias/03_webclient_iis.png](#11-capturas-de-pantalla)*

```powershell
# Exportar el certificado del RD Broker e importarlo en el Web Client
# (usar el certificado autofirmado del servidor en entornos de lab)
Import-RDWebClientBrokerCert "C:\ruta\al\certificado.cer"
```

> En entornos de laboratorio se puede usar el certificado autofirmado generado durante la instalación de RDS. En producción se recomienda un certificado de una CA confiable.

---

### 5.3 Publicar el Web Client

```powershell
Publish-RDWebClientPackage -Type Production -Latest
```

Verificar el acceso desde el navegador:

```
https://<IP-Servidor>/RDWeb/webclient
```

---

## 6. Configuración de IIS y Página Personalizada

### 6.1 Instalar IIS

📸 *Ver captura: [evidencias/01_roles_instalados.png](#11-capturas-de-pantalla)*

1. **Server Manager** → `Add Roles and Features`.
2. Seleccionar rol: **Web Server (IIS)**.
3. Instalar con las características por defecto → **Install**.

---

### 6.2 Crear la página personalizada

📸 *Ver captura: [evidencias/03_webclient_iis.png](#11-capturas-de-pantalla)*

1. Abrir el **Explorador de archivos** → navegar a `C:\inetpub\wwwroot\`.
2. Crear un nuevo archivo `index.html` con el siguiente contenido:

```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>ITLA — Seguridad de Redes</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #1a1a2e;
            color: #e0e0e0;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
        }
        .card {
            background-color: #16213e;
            border: 2px solid #0f3460;
            border-radius: 12px;
            padding: 40px 60px;
            text-align: center;
            box-shadow: 0 0 30px rgba(15, 52, 96, 0.8);
        }
        h1 { color: #e94560; font-size: 2rem; margin-bottom: 10px; }
        h2 { color: #0f3460; font-size: 1.2rem; }
        p  { color: #a8a8b3; margin-top: 20px; }
    </style>
</head>
<body>
    <div class="card">
        <h1>🔐 Seguridad de Redes</h1>
        <h2>Instituto Tecnológico de las Américas — ITLA</h2>
        <p>Arlene Fernández Herrera · Matrícula: 2025-0730</p>
        <p>Página publicada mediante IIS + RDP RemoteApp</p>
    </div>
</body>
</html>
```

3. Guardar el archivo.

---

### 6.3 Verificar IIS en el navegador local

Abrir el navegador en el servidor y navegar a:

```
http://localhost
```

Debe mostrarse la página personalizada con los datos del laboratorio.

---

## 7. Publicar IIS en RDP RemoteApp

### 7.1 Publicar el navegador como RemoteApp apuntando a IIS

📸 *Ver captura: [evidencias/02_remoteapp_coleccion.png](#11-capturas-de-pantalla)*

1. En **Server Manager** → `Remote Desktop Services` → colección `RemoteApp-ITLA`.
2. Clic en `Tasks` → **Publish RemoteApp Programs**.
3. Si el navegador no aparece en la lista automática, clic en **"Add program not listed"**.
4. Ingresar la ruta del ejecutable, por ejemplo:
   - Internet Explorer: `C:\Program Files\Internet Explorer\iexplore.exe`
   - Con argumento: `http://localhost` (para que abra directamente la página IIS)
5. En **Command-line arguments**, agregar: `http://localhost`
6. Publicar.

> Alternativamente, crear un acceso directo `.bat` que lance el navegador hacia `http://localhost` y publicar ese script como RemoteApp.

---

### 7.2 Verificar desde el cliente

Acceder al portal RD Web desde el cliente:

```
https://<IP-Servidor>/RDWeb
```

El icono del navegador/RemoteApp publicado debe aparecer. Al hacer clic, abre la página IIS personalizada en una ventana RemoteApp.

---

## 8. Configuración de NPS (RADIUS Server)

### 8.1 Instalar el rol NPS

📸 *Ver captura: [evidencias/01_roles_instalados.png](#11-capturas-de-pantalla)*

1. **Server Manager** → `Add Roles and Features`.
2. Seleccionar rol: **Network Policy and Access Services**.
3. Seleccionar servicio de rol: ✅ **Network Policy Server (NPS)**.
4. Instalar → **Close**.

---

### 8.2 Registrar NPS en Active Directory

📸 *Ver captura: [evidencias/04_nps_config.png](#11-capturas-de-pantalla)*

1. Abrir **NPS** desde `Tools` → `Network Policy Server`.
2. Clic derecho en el nodo raíz **NPS (Local)** → **Register server in Active Directory**.
3. Confirmar en el cuadro de diálogo.

> Este paso es necesario para que NPS pueda leer los atributos de los usuarios del dominio.

---

### 8.3 Registrar el Router como cliente RADIUS

📸 *Ver captura: [evidencias/04_nps_config.png](#11-capturas-de-pantalla)*

1. En NPS → expandir **RADIUS Clients and Servers** → clic derecho en **RADIUS Clients** → **New**.
2. Configurar:

| Campo | Valor |
|---|---|
| Friendly Name | `Router-Cisco` |
| Address (IP) | IP del router Cisco |
| Shared Secret | `ITLA2025Radius` |
| Confirm Shared Secret | `ITLA2025Radius` |

3. Clic en **OK**.

---

### 8.4 Crear grupos de usuarios en Active Directory

📸 *Ver captura: [evidencias/04_nps_config.png](#11-capturas-de-pantalla)*

Abrir **Active Directory Users and Computers** → `Users` → crear dos grupos:

| Grupo | Descripción |
|---|---|
| `VPN-Nivel15` | Usuarios con acceso privilegiado (nivel 15 — modo enable) |
| `VPN-Nivel1` | Usuarios con acceso de solo lectura (nivel 1) |

Agregar los usuarios correspondientes a cada grupo.

---

### 8.5 Crear política de red — Nivel 15

📸 *Ver captura: [evidencias/04_nps_config.png](#11-capturas-de-pantalla)*

1. En NPS → **Policies** → **Network Policies** → clic derecho → **New**.
2. Nombre de la política: `Acceso-Nivel15`.
3. **Conditions** → Add → `Windows Groups` → agregar `VPN-Nivel15`.
4. **Access Permission**: ✅ **Grant Access**.
5. **Settings** → **RADIUS Attributes** → **Vendor Specific**:
   - Clic en **Add** → seleccionar vendor **Cisco**.
   - Attribute: `cisco-av-pair`
   - Value: `shell:priv-lvl=15`
6. Finalizar.

---

### 8.6 Crear política de red — Nivel 1

📸 *Ver captura: [evidencias/04_nps_config.png](#11-capturas-de-pantalla)*

1. Repetir el proceso anterior con nombre: `Acceso-Nivel1`.
2. **Conditions** → `Windows Groups` → agregar `VPN-Nivel1`.
3. **Settings** → **RADIUS Attributes** → **Vendor Specific**:
   - Vendor: **Cisco**
   - Attribute: `cisco-av-pair`
   - Value: `shell:priv-lvl=1`
4. Finalizar.

> El atributo VSA `shell:priv-lvl=15` indica a IOS que el usuario autenticado debe recibir nivel de privilegio 15 (acceso completo). El valor `1` otorga solo acceso de usuario básico.

---

## 9. Configuración del Router — AAA con RADIUS

### 9.1 Configuración inicial (CLI)

```cisco
! ══════════════════════════════════════════════════════════════
!  Router Cisco — AAA con RADIUS (NPS Windows Server)
!  Arlene Fernández Herrera · 2025-0730 | Seguridad de Redes
! ══════════════════════════════════════════════════════════════

! ── Usuario local de respaldo ────────────────────────────────
username admin privilege 15 secret Admin2025ITLA
username user1 privilege 1 secret User2025ITLA

! ── Habilitar AAA ────────────────────────────────────────────
aaa new-model

! ── Servidor RADIUS (NPS en Windows Server) ─────────────────
radius server NPS-ITLA
 address ipv4 <IP-WindowsServer> auth-port 1812 acct-port 1813
 key ITLA2025Radius

! ── Lista de autenticación: primero RADIUS, luego local ──────
aaa authentication login AUTENTICACION-SSH group radius local
aaa authorization exec AUTORIZACION-SSH group radius local

! ── Contraseña para modo enable (configuración) ──────────────
enable secret Enable2025ITLA

! ── Aplicar AAA a las líneas VTY (SSH) ───────────────────────
line vty 0 4
 login authentication AUTENTICACION-SSH
 authorization exec AUTORIZACION-SSH
 transport input ssh

! ── Configurar SSH ───────────────────────────────────────────
ip domain-name itla.edu.do
crypto key generate rsa modulus 2048
ip ssh version 2

! ── Logs de autenticación, autorización y RADIUS ────────────
! Ejecutar en modo privilegiado para troubleshooting:
! debug aaa authentication
! debug aaa authorization
! debug radius
! show aaa servers
! show aaa sessions
```

---

### 9.2 Verificar estado del servidor RADIUS

```cisco
Router# show aaa servers

RADIUS: id 1, priority 1, host <IP-WindowsServer>, auth-port 1812, acct-port 1813
     State: current UP, duration 120s, previous duration 0s
     Dead: total time 0s, count 0
```

---

### 9.3 Comandos de debug para autenticación

```cisco
! Activar debug completo de AAA y RADIUS
Router# debug aaa authentication
Router# debug aaa authorization
Router# debug radius

! Ver sesiones activas
Router# show aaa sessions

! Ver estado de servidores RADIUS
Router# show aaa servers

! Detener todos los debugs
Router# undebug all
```

Salida esperada al conectarse un usuario nivel 15 por SSH:

```
*AAA/AUTHEN: Method=radius (radius)
*RADIUS: Received from id 1 <IP-NPS>, Access-Accept
*AAA/AUTHOR: Method=radius (radius)
*RADIUS/DECODE: cisco-av-pair = "shell:priv-lvl=15"
*AAA/AUTHOR: Authorization successful - privilege level 15
```

---

## 10. Pruebas desde el Cliente

### 10.1 Acceder mediante RDP RemoteApp Web Client

📸 *Ver captura: [evidencias/05_cliente_rdweb_iis.png](#11-capturas-de-pantalla)*

1. Abrir el navegador en el cliente (VM o máquina Host).
2. Navegar a:
   ```
   https://<IP-Servidor>/RDWeb/webclient
   ```
3. Iniciar sesión con las credenciales de dominio.
4. Hacer clic en la aplicación publicada (navegador con página IIS).
5. Verificar que se abre la página personalizada a través de RemoteApp.

---

### 10.2 Acceder mediante RDP RemoteApp clásico

📸 *Ver captura: [evidencias/06_cliente_rdp_clasico.png](#11-capturas-de-pantalla)*

1. Navegar a:
   ```
   https://<IP-Servidor>/RDWeb
   ```
2. Iniciar sesión con credenciales de dominio.
3. Hacer clic en la aplicación → se descarga un archivo `.rdp`.
4. Abrir el archivo `.rdp` → se abre la aplicación en una ventana RemoteApp.

---

### 10.3 Probar SSH al Router con usuario Nivel 15

📸 *Ver captura: [evidencias/07_ssh_nivel15.png](#11-capturas-de-pantalla)*

```bash
# Desde el cliente (Linux/Windows)
ssh usuario-nivel15@<IP-Router>
```

Resultado esperado: el usuario queda en **prompt `#`** (modo privilegiado nivel 15) directamente.

```
Router#
```

---

### 10.4 Probar SSH al Router con usuario Nivel 1

📸 *Ver captura: [evidencias/08_ssh_nivel1.png](#11-capturas-de-pantalla)*

```bash
ssh usuario-nivel1@<IP-Router>
```

Resultado esperado: el usuario queda en **prompt `>`** (modo usuario nivel 1), sin acceso a comandos privilegiados.

```
Router>
```

---

## 11. Capturas de Pantalla

Todas las capturas se encuentran en la carpeta [`evidencias/`](evidencias/).

| # | Captura | Sección | Descripción |
|---|---|---|---|
| 01 | [Roles instalados](evidencias/01_roles_instalados.png) | §4.1 / §6.1 / §8.1 | Server Manager mostrando los tres roles instalados: Remote Desktop Services, Web Server (IIS) y Network Policy Server. |
| 02 | [Colección RemoteApp + app publicada](evidencias/02_remoteapp_coleccion.png) | §4.2 / §4.3 / §7.1 | Colección `RemoteApp-ITLA` con la aplicación publicada apuntando a `http://localhost`. |
| 03 | [RD Web Client activo + página IIS](evidencias/03_webclient_iis.png) | §5.3 / §6.2 | Portal `https://<IP>/RDWeb/webclient` accesible en el navegador junto a la página IIS personalizada en `http://localhost`. |
| 04 | [NPS — Cliente RADIUS + Políticas](evidencias/04_nps_config.png) | §8.3 / §8.5 / §8.6 | Consola NPS mostrando el cliente RADIUS `Router-Cisco` registrado y las dos políticas `Acceso-Nivel15` y `Acceso-Nivel1` con sus atributos VSA. |
| 05 | [RemoteApp Web Client — página IIS](evidencias/05_cliente_rdweb_iis.png) | §10.1 | Navegador del cliente abriendo la página IIS personalizada a través del portal RD Web Client. |
| 06 | [RemoteApp clásico — página IIS](evidencias/06_cliente_rdp_clasico.png) | §10.2 | Ventana RemoteApp clásica (archivo `.rdp`) mostrando la página IIS personalizada. |
| 07 | [SSH Nivel 15](evidencias/07_ssh_nivel15.png) | §10.3 | Terminal del cliente con sesión SSH activa y prompt `Router#` — nivel de privilegio 15 asignado por RADIUS. |
| 08 | [SSH Nivel 1](evidencias/08_ssh_nivel1.png) | §10.4 | Terminal del cliente con sesión SSH activa y prompt `Router>` — nivel de privilegio 1 asignado por RADIUS. |

---

## 12. Video Demostrativo

🎥 **[Ver en YouTube — enlace pendiente](#)**

**Duración estimada:** < 8 minutos

---

<div align="center">

**Arlene Fernández Herrera · 2025-0730 · ITLA**  
*RDP RemoteApp + IIS + NPS RADIUS + AAA Router*

</div>
