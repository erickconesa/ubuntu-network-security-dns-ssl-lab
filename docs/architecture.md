
# Arquitectura del laboratorio — Semana 2

## Descripción

Durante la segunda semana continué trabajando sobre el servidor Ubuntu Server desplegado y asegurado en la Semana 1.

El objetivo fue ampliar su seguridad incorporando filtrado de tráfico, protección SSH, resolución DNS y comunicaciones HTTPS mediante certificados TLS.

## Infraestructura

- Hipervisor: Hyper-V
- Servidor: Ubuntu Server
- IP privada del servidor: 10.10.30.30
- Red del servidor: 10.10.30.0/24
- Gateway de red: OPNsense (10.10.30.1)
- Máquina de pruebas: Windows 11
- IP del equipo de pruebas: 10.10.30.9

## Servicios

| Servicio | Puerto TCP | Función |
|---|---|---|
| SSH | 48222 | Administración remota |
| HTTP | 80 | Servicio web y redirección |
| HTTPS | 443 | Comunicación cifrada |

## Componentes de seguridad

- UFW: filtrado de tráfico entrante.
- Fail2ban: protección contra intentos repetidos de autenticación SSH.
- Nginx: servidor web.
- Dynu: resolución DNS dinámica.
- DuckDNS: validación del dominio mediante DNS-01.
- Let's Encrypt: emisión de certificados TLS.
- Certbot: gestión y renovación de certificados.

## Publicación HTTPS

Para evitar alterar las reglas NAT existentes de los puertos públicos 80 y 443, se configuró una publicación temporal utilizando el puerto externo 9443, redirigido al puerto 443 del servidor Ubuntu.

Esta publicación estaba prevista únicamente para las comprobaciones del tutor hasta el 31 de octubre de 2026.

## Alcance

Las pruebas del laboratorio se realizaron sobre infraestructura propia y autorizada.

Los resultados de validación desde la red interna no se consideran automáticamente pruebas de accesibilidad desde Internet.
