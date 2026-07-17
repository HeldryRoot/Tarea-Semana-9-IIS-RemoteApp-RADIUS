# Tarea-Semana-9-IIS-RemoteApp-RADIUS

<img width="654" height="276" alt="image" src="https://github.com/user-attachments/assets/edfcc30b-66cc-4465-95dc-13ea7e5fda80" />

**LABORATORIO DE NETWORKING**

**Tarea Semana 9 — IIS, RemoteApp y RADIUS**

_Documentación Técnica Profesional_

**Heldry Terrero**

Matrícula: 2025-0719

Materia: Seguridad de Redes

Fecha: Julio 2026

|   |
|---|
|⚠ AVISO IMPORTANTE — Este proyecto fue desarrollado únicamente con fines educativos en el marco de la asignatura Seguridad de Redes del ITLA. QUEDA PROHIBIDO usar este material en redes de terceros sin autorización.|

  

# 1. Objetivo

Integrar IIS, RD RemoteApp, RD Web Access y NPS/RADIUS con autenticación AAA centralizada en router Cisco. Windows Server 2019 actúa como servidor web, servidor de aplicaciones remotas y servidor RADIUS.

•        Publicar página IIS personalizada.

•        RD RemoteApp — publicar IE como aplicación remota vía .rdp.

•        RD Web Access — portal navegador https://IP/rdweb.

•        Publicar página IIS en el portal RemoteApp.

•        NPS: Nivel_15 (privilege 15) y Nivel_1 (privilege 1).

•        Router con AAA y autenticación RADIUS.

•        Verificar con debug aaa authentication, debug radius, show aaa servers.

# 2. Marco Teórico

## 2.1 RD RemoteApp

Publica aplicaciones del servidor que se ejecutan remotamente y aparecen como ventanas locales en el cliente via RDP (TCP/3389). El usuario descarga un .rdp que abre la aplicación directamente sin ver el escritorio completo del servidor.

## 2.2 RD Web Access — RemoteApp Web Client

Portal HTTPS hospedado en IIS. El usuario accede desde el navegador a https://IP/rdweb, introduce credenciales de dominio y abre las RemoteApps publicadas. No requiere configuración en el cliente. Al hacer clic, el navegador descarga un .rdp firmado que abre la app.

## 2.3 IIS

Servidor web Windows. Publica la página HTML personalizada (http://IP) y hospeda el portal RD Web Access (https://IP/rdweb) simultáneamente.

## 2.4 NPS/RADIUS

NPS implementa RADIUS (RFC 2865, UDP/1812). Para privilege 15 envía Service-Type=Administrative. Para privilege 1 no envía ese atributo. El router asigna el nivel según la respuesta recibida.

|**Componente**|**Protocolo**|**Función**|
|---|---|---|
|IIS|HTTP/80, HTTPS/443|Página personalizada + portal rdweb|
|RD RemoteApp|RDP/3389|Apps publicadas via archivo .rdp|
|RD Web Access|HTTPS/443|Portal navegador https://IP/rdweb|
|NPS/RADIUS|UDP/1812|Autenticación centralizada para el router|
|Cisco AAA|UDP/1812|Cliente RADIUS — delega autenticación SSH|

# 3. Topología

|   |
|---|
<img width="1135" height="521" alt="Topologia Semana 9" src="https://github.com/user-attachments/assets/378623bd-1953-41e5-892e-cc2d9dfe785f" />


|**Dispositivo**|**Interfaz**|**IP**|**Máscara**|**Gateway**|**Función**|
|---|---|---|---|---|---|
|Router|e0/1 LAN|20.25.7.1|255.255.255.0|—|Gateway LAN Usuarios|
|Router|e0/2 DMZ|20.25.19.1|255.255.255.0|—|Gateway DMZ Servidores|
|WinServer-2019|eth0|20.25.19.10|255.255.255.0|20.25.19.1|IIS+NPS+RDS|
|PC-Windows10|eth0|20.25.7.10|255.255.255.0|20.25.7.1|Cliente|

|**Usuario**|**Grupo**|**Contraseña**|**Privilege**|**Acceso al Router**|
|---|---|---|---|---|
|admin15|Nivel_15|Admin123|15|Router-Principal# (modo privilegiado)|
|admin1|Nivel_1|Admin123|1|Router-Principal> (solo lectura)|

# 4. Configuración Router Cisco

enable  
configure terminal  
hostname Router-Principal  
!  
banner motd #  
=============================================================  
  ALERT: RESTRICTED ACCESS - AUTHORIZED PERSONNEL ONLY  
  [ Operator: Heldry Terrero ] [ ID: 2025-0719 ]  
=============================================================  
#  
!  
interface ethernet0/1  
 description LAN_USUARIOS  
 ip address 20.25.7.1 255.255.255.0  
 no shutdown  
!  
interface ethernet0/2  
 description DMZ_SERVIDORES  
 ip address 20.25.19.1 255.255.255.0  
 no shutdown  
!  
username admin privilege 15 secret Admin123  
!  
ip domain-name itla.local  
crypto key generate rsa modulus 1024  
ip ssh version 2  
!  
aaa new-model  
!  
radius server NPS_SERVER  
 address ipv4 20.25.19.10 auth-port 1812 acct-port 1813  
 key Cisco123  
!  
aaa authentication login default group radius local  
aaa authorization exec default group radius local if-authenticated  
!  
enable secret Admin123  
!  
line vty 0 4  
 login authentication default  
 transport input ssh  
 exec-timeout 10 0  
exit  
!  
line console 0  
 login authentication default  
 exec-timeout 10 0  
exit  
!  
do write memory

|   |
|---|
|CRITICO: El shared secret (key Cisco123) debe ser EXACTAMENTE igual en el router y en el RADIUS Client del NPS. Si difieren, Access-Reject aunque las credenciales sean correctas.|

show aaa servers  
show aaa sessions  
!  
debug aaa authentication  
debug aaa authorization  
debug radius  
undebug all

# 5. Configuración IIS — Página Personalizada

Guardar como C:\inetpub\wwwroot\index.html:

<!DOCTYPE html><html lang="es"><head><meta charset="UTF-8">  
<title>ITLA</title><style>body{background:#003399;color:white;font-family:Arial;text-align:center;padding:50px;}  
h1{font-size:48px;}p{font-size:24px;}</style></head><body>  
<h1>ITLA</h1><p>Instituto Tecnológico de Las Américas</p>  
<p><strong>Estudiante:</strong> Heldry Terrero</p>  
<p><strong>Matrícula:</strong> 2025-0719</p>  
<p>Tarea Semana 9 — IIS + RemoteApp + RADIUS</p></body></html>

# 6. RD RemoteApp — Configuración y Publicación

RD RemoteApp permite publicar aplicaciones del servidor que se ejecutan remotamente y aparecen como ventanas locales en el cliente via RDP.

## 6.1 Crear Session Collection

•        Server Manager > Remote Desktop Services > Session Collections > Create Session Collection.

•        Nombre: RemoteApp-ITLA.

•        RD Session Host: WinServer-2019.

•        Usuarios: agregar Nivel_15 y Nivel_1.

## 6.2 Publicar IE como RemoteApp

•        RemoteApp-ITLA > RemoteApp Programs > Publish RemoteApp Programs.

•        Seleccionar Internet Explorer (iexplore.exe) > Publish.

•        En Properties de IE > Command-line arguments: http://20.25.19.10 (carga la página IIS al abrir).

•        IE queda disponible en estado Published.

# 7. RD Web Access — RemoteApp Web Client (Portal Navegador)

RD Web Access publica un portal HTTPS hospedado en IIS desde donde se acceden las RemoteApps sin configuración en el cliente. URL: https://20.25.19.10/rdweb

## 7.1 Configurar RD Web Access

•        Server Manager > Remote Desktop Services > RD Web Access > Configurar.

•        Fuente de RemoteApps: localhost.

•        Verificar en IIS Manager que el sitio RDWeb esté activo.

•        El certificado puede ser autofirmado — el cliente debe aceptar la excepción.  
  

## 7.2 Acceder al Portal desde PC-Windows10

# Navegador en PC-Windows10:  
https://20.25.19.10/rdweb  
   
# Login con credenciales de dominio:  
  Usuario   : DOMINIO\admin15  
  Password  : Admin123  
   
# El portal muestra Internet Explorer como RemoteApp disponible.  
# Al hacer clic: descarga .rdp firmado y abre IE con la página IIS cargada.

# 8. Configuración NPS — RADIUS Server

## 8.1 Pasos de Configuración

•        NPS Console > clic derecho NPS (Local) > Register Server in Active Directory > Yes.

•        RADIUS Clients > New > Friendly Name: Router-Principal, IP: 20.25.7.1, Secret: Cisco123.

•        Active Directory: crear grupos Nivel_15 y Nivel_1 con usuarios admin15 y admin1.

•        En cada usuario: Dial-in > Network Access Permission > Allow Access.

|**Usuario**|**Grupo**|**Contraseña**|**Nivel**|
|---|---|---|---|
|admin15|Nivel_15|Admin123|privilege 15|
|admin1|Nivel_1|Admin123|privilege 1|

## 8.2 Política Nivel 15 (Administrador)

Network Policies > New:  
  Name      : Politica_Nivel_15  
  Conditions: Windows Groups = Nivel_15  
  Access    : Grant access  
  Settings > RADIUS Attributes > Standard:  
    - Eliminar : Framed-Protocol  
    - Agregar  : Service-Type = Administrative  <- privilege 15 en Cisco

|   |
|---|
|Service-Type = Administrative le indica al router Cisco que asigne privilege level 15. Sin este atributo el usuario queda en nivel 1 aunque sea autenticado correctamente.|

| <img width="923" height="349" alt="Network Policy" src="https://github.com/user-attachments/assets/dc270643-cbee-44ec-ad1f-a515a7c47796" />  |



## 8.3 Política Nivel 1 (Básico)

Network Policies > New:  
  Name      : Politica_Nivel_1  
  Conditions: Windows Groups = Nivel_1  
  Access    : Grant access  
  Settings > RADIUS Attributes > Standard:  
    - Eliminar : Framed-Protocol  
    - NO agregar Service-Type  <- privilege 1 por defecto

|   |
|---|
|<img width="923" height="349" alt="Network Policy" src="https://github.com/user-attachments/assets/e04f753c-b952-47ac-b5c1-4d15560a699f" />
|
<img width="681" height="272" alt="Radius Client" src="https://github.com/user-attachments/assets/460525f5-781a-4e58-95d1-b0c8f5448092" />


# 9. Demostración — Verificación Completa

## 9.1 Ping al Servidor

ping 20.25.19.10   # 4 Reply <- routing LAN-DMZ OK

|   |
|---|
<img width="535" height="79" alt="Ping" src="https://github.com/user-attachments/assets/c9d968eb-2576-480c-affd-3f57fd74ded4" />


## 9.2 Página IIS

http://20.25.19.10   # Página Heldry Terrero / 2025-0719

|   |
|---|
<img width="1195" height="896" alt="Servidor IIS" src="https://github.com/user-attachments/assets/a94eb256-aa3f-4251-8f5c-7aabc581bc38" />


## 9.3 RD Web Access + RemoteApp

https://20.25.19.10/rdweb  
# Login admin15/Admin123 -> abrir IE -> carga http://20.25.19.10

|   |
|---|
<img width="1175" height="837" alt="RemoteApp" src="https://github.com/user-attachments/assets/814c8fb9-3de0-41c0-b176-53f2ae26d7f9" />


## 9.4 SSH RADIUS Nivel 15

ssh admin15@20.25.7.1  # Password: Admin123  
Router-Principal#  <- privilege 15 directo

|   |
|---|
<img width="763" height="284" alt="admin15 radius" src="https://github.com/user-attachments/assets/b6f32353-e61a-4b1b-88cf-19f5c53ecd26" />


## 9.5 SSH RADIUS Nivel 1

ssh admin1@20.25.7.1   # Password: Admin123  
Router-Principal>  <- privilege 1

|   |
|---|
<img width="756" height="274" alt="admin1 radius" src="https://github.com/user-attachments/assets/4b3a6bda-4212-4742-bc85-225b38c6b3d6" />


## 9.6 Debugs AAA

Router-Principal# debug aaa authentication  
Router-Principal# debug aaa authorization  
Router-Principal# debug radius  
Router-Principal# show aaa servers  
Router-Principal# show aaa sessions  
Router-Principal# undebug all

# 10. Troubleshooting

|**Síntoma**|**Causa**|**Solución**|
|---|---|---|
|Authentication failed|Shared secret no coincide|Igualar key en router con Shared Secret NPS|
|Queda en > siendo Nivel_15|Falta Service-Type=Administrative|Agregar atributo en Politica_Nivel_15|
|Access-Reject admin1|Dial-in no habilitado|User Properties > Dial-in > Allow Access|
|rdweb error SSL|Cert autofirmado|Aceptar excepción en el navegador|
|SSH connection refused|SSH no configurado|Verificar crypto key, ip ssh v2, transport input ssh|
|Router no llega a NPS|Sin ruta a 20.25.19.10|Verificar ip route o conectividad DMZ|

# 11. Conclusión

La Tarea Semana 9 integra IIS, RD RemoteApp, RD Web Access y NPS/RADIUS con autenticación AAA. El portal rdweb permite acceder a la página IIS como RemoteApp desde el navegador. Los usuarios de Nivel_15 acceden directamente a Router-Principal# via el atributo Service-Type=Administrative, mientras Nivel_1 obtiene acceso básico. Los debugs AAA confirman el funcionamiento completo de la cadena RADIUS.

|   |
|---|
|RESULTADO: IIS OK. RD RemoteApp publicado. RD Web Access en https://20.25.19.10/rdweb. Página IIS en RemoteApp. SSH Nivel_15 = privilege 15. SSH Nivel_1 = privilege 1. Debugs AAA verificados.|

Heldry Terrero — Matrícula: 2025-0719 — Seguridad de Redes — Julio 2026

Link del Video:  [https://youtu.be/deFIMX8JAQg](https://youtu.be/deFIMX8JAQg)  
  

Link del GitHub: [https://github.com/HeldryRoot/Tarea-Semana-9-IIS-RemoteApp-RADIUS](https://github.com/HeldryRoot/Tarea-Semana-9-IIS-RemoteApp-RADIUS)
