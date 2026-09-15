# Alternativas a Spotify — Streaming personal con Navidrome

Repositorio de apoyo para el taller **"Alternativas a Spotify: relatorio de xestión multimedia con software libre"**, impartido por Alberto Permuy Leal (Comunidade O Zulo).

El taller plantea recuperar el control de tu biblioteca musical montando tu propio servidor de streaming personal, en lugar de depender de plataformas como Spotify.


## Contenido del repositorio

```
.
├── docker/
│   ├── docker-compose.yml
│   └── ...
├── presentacion/
│   └── RELATORIO_SPOTIFY_2026.pdf
└── README.md
```

- **`docker/`** — Despliegue completo del stack de streaming personal (Navidrome, NFS y Nginx Proxy Manager).
- **`presentacion/`** — Diapositivas del taller en PDF.

## La presentación

El relatorio recorre:

- **Qué es Spotify y cómo funciona** a nivel técnico: microservicios en Java/Python, infraestructura en Google Cloud, frontend en React/Redux/Webpack, backend con PostgreSQL y Cassandra, todo orquestado con Docker + Terraform + Kubernetes — y construido, en buena parte, sobre software libre.
- **Cómo genera ingresos**: suscripciones Premium, publicidad, acuerdos con terceros y el "Spotify Partner Program".
- **El mito del pago por stream**: Spotify no paga por reproducción de forma directa; entran en juego ubicación, dispositivo, círculo de amigos y precio.
- **Qué datos recoge de ti**: desde datos de cuenta (nombre, email, fecha de nacimiento, género, país) hasta historial de búsquedas y reproducciones, interacciones con anuncios, dirección IP, dispositivos conectados, ubicación aproximada o exacta, y grabaciones de voz cuando usas esas funciones — todo documentado en su política de privacidad (junio 2026).
- **La alternativa DIY**: montar tu propio servidor con hardware reacondicionado, Raspberry Pi o NAS, usando Navidrome como reemplazo funcional.
- **Qué ganas y qué pierdes** con el cambio — control, privacidad e independencia, a cambio de catálogo inmediato y comodidad.

## Arquitectura

El stack se despliega en un servidor Debian con Docker, y expone el servicio hacia fuera vía HTTPS:

```
Clientes (navegador, Substreamer, Feishin)
        │
        ▼
     Internet
        │
        ▼
   DuckDNS (DNS dinámico)
        │
        ▼  HTTPS / SSL-TLS
┌─────────────────────────────┐
│   Servidor Debian + Docker   │
│                               │
│  Nginx Proxy Manager ◄──────┼── DuckDNS actualiza la IP pública
│         │                    │
│         ▼                    │
│     Navidrome ◄──────────────┼── NFS: biblioteca MP3 compartida
└─────────────────────────────┘
```

- **Nginx Proxy Manager** recibe las conexiones HTTPS y las redirige a Navidrome, gestionando los certificados SSL/TLS.
- **Navidrome** accede a la biblioteca de música a través de NFS.
- Clientes compatibles: navegador web, apps móviles tipo Substreamer, o clientes de escritorio como Feishin (vía protocolo Subsonic/OpenSubsonic).

## Stack Docker

El `docker-compose.yml` en `docker/` despliega tres servicios:

### `navidrome`
Servidor de streaming musical (imagen `deluan/navidrome:latest`), expuesto en el puerto `4533`.

- Lee la librería musical en modo solo lectura.
- Escaneo automático de la librería vía cron (`ND_SCANNER_SCHEDULE`).
- Sistema de plugins habilitado (`ND_PLUGINS_ENABLED`).
- Backups automáticos y rotativos de la base de datos (usuarios, playlists, historial) mediante `ND_BACKUP_PATH`, `ND_BACKUP_SCHEDULE` y `ND_BACKUP_COUNT` — **esto respalda el estado de la aplicación, no los ficheros de audio**.

### `nfs-server`
Servidor NFS containerizado (`itsthenetwork/nfs-server-alpine`) que exporta la carpeta de música en modo solo lectura, para poder montarla como recurso de red desde otros equipos.

> Requiere `privileged: true`: el servidor NFS necesita acceso a módulos del kernel del host, es una limitación inherente a containerizar NFS.

### `npm`
Nginx Proxy Manager (`jc21/nginx-proxy-manager`), para el reverse proxy y la gestión de certificados Let's Encrypt vía panel web.

## Puesta en marcha

1. Clona el repositorio y entra en `docker/`.
2. Ajusta en `docker-compose.yml`:
   - La ruta de tu librería musical (bind mounts de `navidrome` y `nfs-server`).
   - `user: "UID:GID"` de Navidrome al usuario propietario de esa carpeta (verifica con `id <usuario>`).
   - `PERMITTED` en `nfs-server`, con la IP o rango de tu red que podrá montar el share.
   - `ND_BASEURL` con tu dominio.
3. Levanta el stack:
   ```bash
   docker compose up -d
   ```
4. Entra al panel de Nginx Proxy Manager (`http://<IP-del-host>:81`) y configura un *Proxy Host* apuntando a `navidrome:4533`, con SSL vía Let's Encrypt.
5. Accede a Navidrome desde tu dominio o ip configurada.

---

*Alberto Permuy Leal — Comunidade O Zulo*
