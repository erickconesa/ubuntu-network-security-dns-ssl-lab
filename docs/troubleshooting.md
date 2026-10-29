
# Resolución de incidencias — Semana 2

## Introducción

Durante la segunda semana de prácticas en CorGon CyberSolutions surgieron varias incidencias relacionadas con el firewall, la publicación de servicios web, la resolución DNS y la automatización de certificados TLS.

Este documento recoge los problemas que encontré, cómo identifiqué sus causas y las soluciones que apliqué en mi laboratorio.

---

## INC-01 — Puertos permitidos por UFW que aparecían cerrados

### Problema

Después de configurar UFW, observé que los puertos 80 y 443 no aparecían abiertos en el escaneo inicial realizado con Nmap desde Windows 11.

Sin embargo, las reglas del firewall permitían expresamente el tráfico hacia ambos puertos.

### Diagnóstico

Comprobé las reglas con:

```bash
sudo ufw status verbose
```

Después revisé los servicios en escucha:

```bash
sudo ss -tulpn
```

La comprobación mostró que todavía no había ningún servidor web escuchando en los puertos 80 y 443.

### Solución

Instalé y configuré Nginx, habilitando los servicios HTTP y HTTPS.

Repetí el escaneo:

```bash
nmap -Pn -p- 10.10.30.30
```

### Resultado

El escaneo final mostró los puertos 80, 443 y 48222/TCP abiertos y los otros 65.532 puertos TCP filtrados.

### Aprendizaje

Comprendí que autorizar un puerto en el firewall no significa que exista un servicio escuchando en él.

---

## INC-02 — El servidor web no era accesible desde Internet

### Problema

Después de configurar Nginx intenté acceder a la página desde una conexión móvil externa, pero la petición agotaba el tiempo de espera.

### Diagnóstico

Revisé la topología de red de mi laboratorio:

Vodafone → Tenda (WISP) → OPNsense → Red del servidor Ubuntu

Identifiqué que OPNsense ya utilizaba las redirecciones de los puertos públicos 80 y 443 para otro servidor ubicado en la DMZ.

Modificar esas reglas podía afectar al funcionamiento de los servicios existentes.

### Solución

Decidí mantener las reglas NAT preexistentes y utilizar el desafío DNS-01 de Let's Encrypt para emitir el certificado TLS sin necesitar que la autoridad certificadora accediera al servidor por HTTP.

Posteriormente habilité una publicación HTTPS temporal para el tutor:

- Puerto público: 9443/TCP.
- Puerto interno: 443/TCP.
- Destino: servidor Ubuntu.

### Resultado

Conservé los servicios ya publicados en los puertos habituales y preparé una vía independiente para la comprobación HTTPS del laboratorio.

La publicación temporal estaba prevista hasta el 31 de octubre de 2026.

### Aprendizaje

Aprendí a analizar las reglas NAT antes de modificar una publicación de servicios y a buscar alternativas que no interfieran con el resto de la infraestructura.

---

## INC-03 — Limitaciones de Dynu para el desafío DNS-01

### Problema

Había configurado correctamente un registro A y un CNAME utilizando Dynu.

Sin embargo, al preparar la emisión del certificado mediante DNS-01 encontré restricciones para crear los registros TXT necesarios en nuevos subdominios.

### Diagnóstico

El desafío DNS-01 requiere publicar un registro TXT específico para demostrar el control del dominio.

Los registros A y CNAME configurados inicialmente no sustituían ese requisito.

### Solución

Configuré un dominio adicional mediante DuckDNS, que permitía gestionar el desafío DNS-01 utilizando su API.

Preparé dos scripts Python:

- `auth.py`: publicación y comprobación del registro TXT.
- `cleanup.py`: retirada del registro TXT.

También protegí el token de DuckDNS en un archivo accesible exclusivamente por root.

### Resultado

Conseguí automatizar el desafío DNS-01 y obtener un certificado real de Let's Encrypt para el dominio de DuckDNS.

### Aprendizaje

Comprendí que la resolución DNS normal y la validación de propiedad de un dominio mediante DNS-01 son procesos diferentes.

---

## INC-04 — Error de ejecución en el script Python de autenticación

### Problema

Durante la primera prueba del hook de autenticación de Certbot apareció un error de Python:

```text
UnboundLocalError
```

### Diagnóstico

Revisé el script `auth.py` y detecté un problema de sangría que afectaba a la ejecución de su lógica.

Antes de repetir la prueba comprobé la compilación sintáctica del archivo:

```bash
sudo python3 -m py_compile /etc/letsencrypt/duckdns/auth.py
```

### Solución

Corregí la sangría del script y ejecuté nuevamente la simulación de Certbot.

### Resultado

La segunda prueba `--dry-run` terminó correctamente.

El script publicó el registro TXT, comprobó su propagación mediante resolutores públicos y ejecutó la limpieza posterior.

Después conseguí emitir el certificado real.

### Aprendizaje

Aprendí la importancia de validar los scripts antes de incorporarlos a procesos automatizados y de interpretar las excepciones de Python para encontrar su causa.

---

## INC-05 — Comprobación de la renovación y recarga de Nginx

### Situación

Una vez obtenido el certificado TLS, necesitaba asegurar que su renovación pudiera ejecutarse automáticamente.

Tener instalado Certbot no demostraba por sí solo que la renovación funcionara.

### Diagnóstico

Comprobé el temporizador systemd:

```bash
systemctl status certbot.timer --no-pager
```

También revisé la configuración de renovación para confirmar que conservaba los hooks de autenticación y limpieza de DuckDNS.

### Solución

Ejecuté una renovación simulada:

```bash
sudo certbot renew --dry-run \
  --cert-name ubuntu-erick-asir.duckdns.org
```

Después preparé un deploy hook que comprueba la sintaxis de Nginx y recarga el servicio cuando la renovación del certificado termina correctamente.

Validé su integración con:

```bash
sudo certbot renew --dry-run \
  --run-deploy-hooks \
  --cert-name ubuntu-erick-asir.duckdns.org
```

### Resultado

Las simulaciones finalizaron correctamente y Nginx continuó activo, respondiendo por HTTPS.

No se observó todavía una renovación real futura por vencimiento del certificado.

### Aprendizaje

Comprendí que automatizar un proceso implica comprobar tanto su programación como sus dependencias y acciones posteriores.

---

## Conclusiones

Las incidencias de esta semana me permitieron reforzar conocimientos de administración Linux, redes, DNS, certificados digitales y automatización.

Aprendí a distinguir los problemas de filtrado de tráfico de los problemas de servicios, a revisar la arquitectura NAT antes de publicar aplicaciones y a utilizar DNS-01 cuando la validación HTTP no resulta adecuada.

También mejoré mi capacidad para depurar scripts Python y realizar pruebas controladas antes de dar una configuración por validada.

El objetivo no fue únicamente conseguir que los servicios funcionaran, sino comprender por qué se producían los errores y verificar técnicamente las soluciones aplicadas.

---

**Autor:** Erick Conesa Pérez  
**Ciclo:** Administración de Sistemas Informáticos en Red (ASIR)  
**Prácticas:** CorGon CyberSolutions  
**Semana 2 — Octubre de 2026**
