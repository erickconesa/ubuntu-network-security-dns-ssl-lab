
# 🛡️ Laboratorio de Seguridad de Red, DNS y SSL/TLS — Semana 2

![Ubuntu](https://img.shields.io/badge/Ubuntu-Server-E95420?logo=ubuntu&logoColor=white)
![UFW](https://img.shields.io/badge/Firewall-UFW-blue)
![Fail2ban](https://img.shields.io/badge/IPS-Fail2ban-red)
![Nginx](https://img.shields.io/badge/Web-Nginx-009639?logo=nginx&logoColor=white)
![TLS](https://img.shields.io/badge/TLS-Let's%20Encrypt-green)
![Estado](https://img.shields.io/badge/Estado-Completado-brightgreen)

## 📌 Descripción del proyecto

Este repositorio documenta la segunda semana de mis prácticas del ciclo formativo de Grado Superior en Administración de Sistemas Informáticos en Red (ASIR), realizadas en CorGon CyberSolutions.

El objetivo fue continuar asegurando el servidor Ubuntu desplegado durante la primera semana, incorporando protección perimetral, prevención de ataques de fuerza bruta, resolución DNS, servicios web cifrados mediante HTTPS y renovación automática de certificados digitales.

Todas las configuraciones se realizaron en un entorno de laboratorio propio, verificando el funcionamiento de los servicios mediante comandos, pruebas de conectividad y evidencias técnicas.

**Proyecto anterior:** [Semana 1 — Hardening de Ubuntu Server](https://github.com/erickconesa/ubuntu-server-hardening-lab)

---

## 🎯 Objetivos

- Aplicar una política restrictiva de firewall con UFW.
- Reducir la superficie de exposición del servidor.
- Proteger SSH frente a ataques de fuerza bruta con Fail2ban.
- Configurar y verificar registros DNS A y CNAME.
- Instalar Nginx y habilitar comunicaciones HTTPS.
- Obtener certificados TLS de Let's Encrypt mediante DNS-01.
- Automatizar la renovación de certificados con Certbot.
- Validar los resultados con pruebas reales y documentar las incidencias.

## 🖥️ Entorno de laboratorio

| Componente | Tecnología |
|---|---|
| Servidor | Ubuntu Server |
| Virtualización | Hyper-V y entorno virtualizado |
| Firewall del servidor | UFW |
| Protección de SSH | Fail2ban |
| Servidor web | Nginx |
| DNS dinámico | Dynu y DuckDNS |
| Certificados TLS | Let's Encrypt |
| Gestión de certificados | Certbot |
| Pruebas | Windows 11, Nmap, dig, curl y OpenSSL |
| Firewall de red | OPNsense |

El servidor mantiene la configuración de acceso SSH mediante claves Ed25519 de la primera semana, utilizando un puerto personalizado.

---

## 🔥 CT-01 — Firewall perimetral con UFW

### Trabajo realizado

Comencé comprobando los servicios que estaban escuchando en el servidor.

Configuré UFW con una política de denegación por defecto del tráfico entrante y permiso para las conexiones salientes.

Únicamente autoricé los puertos necesarios para la administración remota y los servicios web.

### Comandos principales

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow 48222/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

sudo ufw enable
sudo ufw status verbose
sudo ss -tulpn
```

### Validación

Realicé un escaneo completo desde Windows 11:

```bash
nmap -Pn -p- 10.10.30.30
```

El resultado final, con Nginx instalado, mostró:

- 80/TCP: abierto.
- 443/TCP: abierto.
- 48222/TCP: abierto.
- Los otros 65.532 puertos TCP: filtrados.

**Aprendizaje:** permitir un puerto en el firewall no implica que exista un servicio escuchando en él.

---

## 🚨 CT-02 — Protección SSH mediante Fail2ban

### Trabajo realizado

Instalé Fail2ban para detectar intentos fallidos de autenticación SSH y bloquear temporalmente las direcciones IP responsables.

Creé una configuración específica para la jail `sshd`, evitando modificar directamente los archivos originales del paquete.

### Configuración

```ini
[sshd]
enabled = true
port = 48222
backend = systemd
maxretry = 5
findtime = 600
bantime = 900
```

Esta configuración establece un máximo de cinco intentos fallidos durante diez minutos y un bloqueo temporal de quince minutos.

### Comprobaciones

```bash
sudo fail2ban-client -t
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

### Prueba práctica

Desde Windows 11 realicé cinco intentos fallidos de autenticación contra el servidor.

Fail2ban detectó los fallos y bloqueó la dirección IP de origen.

Posteriormente retiré el bloqueo manualmente y comprobé que el acceso legítimo mediante clave SSH volvía a funcionar.

**Aprendizaje:** no basta con que una herramienta de seguridad esté activa; es importante verificar que realmente detecta y bloquea los intentos fallidos.

---

## 🌐 CT-03 — Configuración DNS

### Trabajo realizado

Configuré un dominio dinámico con Dynu y creé dos registros DNS:

- **Registro A:** asocia el dominio principal con la dirección IPv4 pública.
- **Registro CNAME:** permite utilizar `www` como alias del dominio principal.

Establecí un TTL de 120 segundos para facilitar la actualización de los registros en un entorno con IP dinámica.

### Validación

```bash
dig ubuntu-erick.giize.com
dig www.ubuntu-erick.giize.com CNAME
```

Ambas consultas devolvieron respuestas correctas, confirmando el funcionamiento de los registros configurados.

**Aprendizaje:** comprendí la diferencia entre un registro A y un CNAME, así como la utilidad del DNS dinámico cuando la dirección IP pública puede cambiar.

---

## 🔐 CT-04 — Nginx y certificados SSL/TLS

### Trabajo realizado

Instalé Nginx, preparé una página de prueba y configuré un Virtual Host para servir contenido web.

```bash
sudo systemctl status nginx
sudo nginx -t
sudo ss -tlnp
```

Durante las pruebas detecté que los puertos públicos 80 y 443 ya estaban siendo utilizados por otro servidor de mi laboratorio, a través de OPNsense.

Para evitar modificar esos servicios decidí utilizar el desafío **DNS-01** de Let's Encrypt.

### Implementación

Configuré un dominio adicional mediante DuckDNS y preparé dos scripts Python:

- `auth.py`: publica y comprueba el registro TXT del desafío DNS-01.
- `cleanup.py`: elimina el registro TXT una vez completada la validación.

Las credenciales de la API se almacenaron en un archivo protegido, accesible únicamente por root.

Tras corregir un error de sangría en el script de autenticación, conseguí superar la validación de prueba y emitir un certificado real de Let's Encrypt.

### Verificación HTTPS

```bash
curl -vI \
  --resolve ubuntu-erick-asir.duckdns.org:443:127.0.0.1 \
  https://ubuntu-erick-asir.duckdns.org
```

La prueba confirmó:

- Negociación TLS 1.3.
- Certificado emitido por Let's Encrypt.
- Coincidencia del nombre del dominio.
- Verificación correcta de la confianza del certificado.
- Respuesta HTTP 200 OK.

También habilité una publicación HTTPS temporal a través del puerto público 9443 para permitir la comprobación externa por parte del tutor, sin sustituir los servicios existentes en los puertos públicos 80 y 443.

**Aprendizaje:** DNS-01 permite demostrar el control de un dominio sin necesitar publicar el servidor web en los puertos HTTP/HTTPS habituales.

---

## 🔄 CT-05 — Renovación automática de certificados

### Trabajo realizado

Comprobé la programación de renovación automática de Certbot y la persistencia de los scripts de autenticación DNS-01.

### Validación del temporizador

```bash
systemctl status certbot.timer --no-pager
systemctl list-timers --all | grep certbot
```

### Prueba de renovación

```bash
sudo certbot renew --dry-run \
  --cert-name ubuntu-erick-asir.duckdns.org
```

La simulación finalizó correctamente.

También preparé un deploy hook para comprobar la configuración de Nginx y recargar el servicio después de una renovación satisfactoria.

```bash
sudo certbot renew --dry-run \
  --run-deploy-hooks \
  --cert-name ubuntu-erick-asir.duckdns.org
```

La prueba integrada terminó correctamente y Nginx continuó respondiendo mediante HTTPS.

**Aprendizaje:** automatizar la renovación requiere tanto una tarea programada como un mecanismo de validación del dominio que pueda ejecutarse sin intervención manual.

La renovación se ha validado mediante simulaciones; todavía no se ha observado una renovación real futura por vencimiento del certificado.

---

## 🧪 Resultados

| Control | Resultado |
|---|---|
| CT-01 — UFW | Validado |
| CT-02 — Fail2ban | Validado |
| CT-03 — DNS | Validado |
| CT-04 — Nginx y TLS | Validado |
| CT-05 — Renovación | Validado |

**Total: 5 controles técnicos validados y 15 evidencias documentadas.**

Las comprobaciones se realizaron en un entorno de laboratorio controlado. Los resultados de los escaneos desde Windows 11 describen la exposición observada desde esa máquina, sin atribuirlos automáticamente al acceso desde Internet.

---

## 🧠 Incidencias y soluciones

Durante esta semana tuve que resolver varias situaciones reales:

**1. Puertos permitidos pero cerrados**

Al principio Nmap no detectaba HTTP y HTTPS como abiertos porque Nginx todavía no estaba instalado. Aprendí a diferenciar las reglas del firewall de los servicios realmente disponibles.

**2. Publicación web y reglas NAT existentes**

Los puertos públicos 80 y 443 ya estaban asignados a otro servidor. Opté por mantener los servicios existentes y utilizar DNS-01 para emitir el certificado.

**3. Limitaciones de registros TXT**

Encontré dificultades para crear los registros TXT necesarios en Dynu. Lo solucioné utilizando DuckDNS para la validación ACME.

**4. Error en el script Python**

La primera prueba del hook de autenticación falló por un error de sangría. Revisé el código, corregí el problema y repetí la validación con éxito.

**5. Renovación automática**

No me limité a verificar que el temporizador existía. Ejecuté renovaciones simuladas y comprobé la integración del deploy hook con Nginx.

---

## 📚 Competencias adquiridas

- Administración de firewalls Linux.
- Reducción de superficie de exposición.
- Detección y mitigación de ataques de fuerza bruta.
- Administración de registros DNS.
- Configuración de servidores web Nginx.
- Gestión de certificados X.509 y conexiones TLS.
- Automatización de procesos mediante scripts Python y systemd.
- Diagnóstico de incidencias de red, DNS y servicios.
- Verificación técnica y documentación de evidencias.

---


## 📁 Documentación técnica

La documentación del laboratorio está organizada en tres archivos:

- [Arquitectura del laboratorio](docs/architecture.md)
- [Validación de los cinco controles técnicos](docs/validation.md)
- [Incidencias y soluciones aplicadas](docs/troubleshooting.md)

## 📸 Evidencias técnicas

Durante la segunda semana se documentaron 15 evidencias reales, organizadas en cinco controles técnicos.

### CT-01 — Firewall UFW

**Política de firewall y servicios expuestos**

![UFW activo](screenshots/ct01-firewall/ev-01-1-ufw-status.png)

![Puertos en escucha](screenshots/ct01-firewall/ev-01-2-listening-ports.png)

![Escaneo Nmap](screenshots/ct01-firewall/ev-01-3-nmap-scan.png)

### CT-02 — Fail2ban

**Servicio activo y bloqueo real por intentos fallidos**

![Servicio Fail2ban](screenshots/ct02-fail2ban/ev-02-1-fail2ban-service.png)

![Jail SSH](screenshots/ct02-fail2ban/ev-02-2-sshd-jail.png)

![Bloqueo de IP](screenshots/ct02-fail2ban/ev-02-3-ssh-ip-ban.png)

### CT-03 — DNS

**Configuración y resolución de registros A y CNAME**

![Registros Dynu](screenshots/ct03-dns/ev-03-1-dynu-records.png)

![Validación registro A](screenshots/ct03-dns/ev-03-2-dig-a-record.png)

![Validación CNAME](screenshots/ct03-dns/ev-03-3-dig-cname-record.png)

### CT-04 — Nginx y SSL/TLS

**Configuración web, emisión y validación del certificado**

![Virtual Host Nginx](screenshots/ct04-tls/ev-04-1-nginx-virtual-host.png)

![Certificado Let's Encrypt](screenshots/ct04-tls/ev-04-2-letsencrypt-certificate.png)

![Validación HTTPS](screenshots/ct04-tls/ev-04-3-https-tls-validation.png)

### CT-05 — Renovación automática

**Certbot, renovación simulada y continuidad HTTPS**

![Temporizador Certbot](screenshots/ct05-renewal/ev-05-1-certbot-timer.png)

![Renovación simulada](screenshots/ct05-renewal/ev-05-2-certbot-dry-run.png)

![Recarga Nginx](screenshots/ct05-renewal/ev-05-3-nginx-reload-https.png)

---

### Resultado final

- 5 controles técnicos validados.
- 15 evidencias documentadas.
- Firewall restrictivo y protección frente a fuerza bruta.
- Resolución DNS verificada.
- Certificado SSL/TLS emitido correctamente.
- Renovación automática validada mediante simulación.

**Semana 2 completada — Octubre de 2026.**


---

## 👨‍💻 Autor

**Erick Conesa Pérez**

Estudiante de Administración de Sistemas Informáticos en Red (ASIR).

Prácticas formativas en CorGon CyberSolutions.

**Semana 2 — Octubre de 2026**

[Consultar laboratorio de la Semana 1](https://github.com/erickconesa/ubuntu-server-hardening-lab)
