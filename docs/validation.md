
# Validación técnica — Semana 2

## Objetivo

Este documento recoge las comprobaciones realizadas durante la segunda semana de prácticas de ASIR en CorGon CyberSolutions.

Las pruebas se ejecutaron en un laboratorio propio, utilizando Ubuntu Server como servidor principal y Windows 11 como equipo de verificación.

Se validaron cinco controles técnicos relacionados con seguridad de red, prevención de intrusiones, DNS, SSL/TLS y automatización de certificados.

---

## CT-01 — Validación del firewall UFW

### Comprobación de la política de seguridad

```bash
sudo ufw status verbose
```

**Resultado obtenido:**

- Firewall activo.
- Política de entrada: `deny`.
- Política de salida: `allow`.
- Excepciones TCP: 48222, 80 y 443.

### Verificación de servicios en escucha

```bash
sudo ss -tulpn
```

Se comprobó inicialmente que SSH escuchaba en el puerto 48222 y que los servicios DNS locales estaban asociados a loopback.

Antes de instalar Nginx, los puertos 80 y 443 estaban permitidos por UFW, pero no tenían ningún servicio web escuchando.

### Escaneo final con Nmap

Desde Windows 11:

```bash
nmap -Pn -p- 10.10.30.30
```

**Resultado final:**

| Estado | Puertos |
|---|---|
| Abiertos | 80, 443 y 48222/TCP |
| Filtrados | 65.532 puertos TCP |

El escaneo confirmó la superficie de exposición del servidor desde Windows 11 después de instalar Nginx.

---

## CT-02 — Validación de Fail2ban

### Comprobación del servicio

```bash
sudo systemctl status fail2ban
sudo fail2ban-client status sshd
```

**Resultado obtenido:**

- Servicio `active (running)`.
- Arranque automático habilitado.
- Jail `sshd` activa.
- Monitorización mediante el journal de systemd.

### Parámetros configurados

| Parámetro | Valor |
|---|---|
| Puerto SSH | 48222 |
| maxretry | 5 |
| findtime | 600 segundos |
| bantime | 900 segundos |

### Prueba de bloqueo real

Desde Windows 11 se realizaron cinco intentos fallidos de autenticación SSH con un usuario inexistente.

Posteriormente se ejecutó:

```bash
sudo fail2ban-client status sshd
```

**Resultado obtenido:**

- `Total failed: 5`
- `Currently banned: 1`
- `Total banned: 1`
- IP bloqueada: `10.10.30.9`

Se confirmó que Fail2ban bloqueó correctamente la dirección IP desde la que se realizaron los intentos fallidos.

Tras retirar el bloqueo, se verificó nuevamente el acceso legítimo mediante clave SSH Ed25519.

---

## CT-03 — Validación de DNS

### Registro A

```bash
dig ubuntu-erick.giize.com
```

**Resultado:**

- Estado DNS: `NOERROR`.
- Resolución hacia la dirección IPv4 pública configurada.

### Registro CNAME

```bash
dig www.ubuntu-erick.giize.com CNAME
```

**Resultado:**

- Estado DNS: `NOERROR`.
- Alias `www.ubuntu-erick.giize.com`.
- Destino `ubuntu-erick.giize.com`.

Los dos registros quedaron configurados con TTL de 120 segundos.

La resolución se comprobó desde Ubuntu Server mediante consultas independientes.

---

## CT-04 — Validación de Nginx y SSL/TLS

### Comprobación del servidor web

```bash
sudo systemctl status nginx --no-pager
sudo nginx -t
sudo ss -tlnp
```

**Resultados:**

- Nginx activo.
- Sintaxis de configuración correcta.
- Puertos 80 y 443 en escucha.

### Validación DNS-01

Se utilizaron scripts Python para publicar y retirar automáticamente el registro TXT de validación de Let's Encrypt mediante DuckDNS.

Después de corregir un error de sangría en el script de autenticación, la simulación se completó correctamente.

El certificado real fue emitido para:

`ubuntu-erick-asir.duckdns.org`

Fecha de caducidad del certificado emitido: **24/01/2027**.

### Comprobación HTTPS

```bash
curl -vI \
  --resolve ubuntu-erick-asir.duckdns.org:443:127.0.0.1 \
  https://ubuntu-erick-asir.duckdns.org
```

**Resultados:**

- Protocolo TLS 1.3.
- Cifrado `TLS_AES_256_GCM_SHA384`.
- Emisor Let's Encrypt.
- Nombre del dominio coincidente.
- Verificación de confianza correcta.
- Respuesta `HTTP/1.1 200 OK`.

Esta prueba confirmó el funcionamiento HTTPS local del servidor.

### Publicación temporal

El 27 de octubre de 2026 se habilitó una redirección NAT del puerto público 9443 al puerto 443 del servidor Ubuntu.

Se preparó una página de verificación para el tutor, manteniendo las publicaciones preexistentes de los puertos públicos 80 y 443.

La retirada del acceso temporal estaba prevista para el 31 de octubre de 2026.

---

## CT-05 — Validación de renovación automática

### Temporizador de Certbot

```bash
systemctl status certbot.timer --no-pager
systemctl list-timers --all | grep certbot
```

**Resultado:**

- Temporizador habilitado.
- Estado `active (waiting)`.
- Servicio de renovación programado mediante systemd.

### Renovación simulada

```bash
sudo certbot renew --dry-run \
  --cert-name ubuntu-erick-asir.duckdns.org
```

**Resultado:**

`Congratulations, all simulated renewals succeeded`

La simulación confirmó que los scripts DNS-01 podían publicar, verificar y retirar automáticamente el registro TXT requerido.

### Validación del deploy hook

```bash
sudo certbot renew --dry-run \
  --run-deploy-hooks \
  --cert-name ubuntu-erick-asir.duckdns.org
```

La prueba integrada completó satisfactoriamente la renovación simulada y ejecutó el deploy hook de Nginx.

### Comprobación final

```bash
systemctl is-active nginx
```

La comprobación posterior confirmó que Nginx continuaba activo y que el servicio HTTPS seguía respondiendo con código HTTP 200.

**Limitación:** se validó el procedimiento de renovación mediante simulación, pero no se observó todavía una renovación real futura del certificado.

---

## Resumen de validaciones

| Control | Estado | Evidencias |
|---|---|---|
| CT-01 — Firewall | Validado | 3 |
| CT-02 — Fail2ban | Validado | 3 |
| CT-03 — DNS | Validado | 3 |
| CT-04 — SSL/TLS | Validado | 3 |
| CT-05 — Renovación | Validado | 3 |
| **Total** | **5/5 validados** | **15** |

## Conclusión

Las pruebas realizadas permitieron comprobar el funcionamiento de los cinco controles técnicos de la segunda semana.

El laboratorio combina configuración de servicios, protección frente a accesos no autorizados, cifrado de comunicaciones y automatización.

Las evidencias demuestran los resultados observados dentro del alcance de cada prueba, diferenciando las validaciones locales, las realizadas desde Windows 11 y la publicación HTTPS temporal para comprobación externa.

---

**Autor:** Erick Conesa Pérez  
**Formación:** Administración de Sistemas Informáticos en Red (ASIR)  
**Empresa de prácticas:** CorGon CyberSolutions  
**Periodo:** Octubre de 2026
