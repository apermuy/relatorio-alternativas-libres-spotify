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

- **`docker/`** — Despliegue completo del stack de streaming personal (Navidrome, NFS, Nginx Proxy Manager y DuckDNS).
- **`presentacion/`** — Diapositivas del taller en PDF.

## La presentación

El relatorio recorre:

- **Qué es Spotify y cómo funciona** a nivel técnico: microservicios en Java/Python, infraestructura en Google Cloud, frontend en React/Redux/Webpack, backend con PostgreSQL y Cassandra, todo orquestado con Docker + Terraform + Kubernetes — y construido, en buena parte, sobre software libre.
- **Cómo genera ingresos**: suscripciones Premium, publicidad, acuerdos con terceros y el "Spotify Partner Program".
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

- **DuckDNS** mantiene actualizada la IP pública asociada a tu dominio, útil si tu conexión no tiene IP fija.
- **Nginx Proxy Manager** recibe las conexiones HTTPS y las redirige a Navidrome, gestionando los certificados SSL/TLS.
- **Navidrome** accede a la biblioteca de música a través de NFS.
- Clientes compatibles: navegador web, apps móviles tipo Substreamer, o clientes de escritorio como Feishin (vía protocolo Subsonic/OpenSubsonic).

## Requisitos previos

- Docker y Docker Compose instalados en el servidor.
- Un usuario del sistema propietario de la carpeta de música, con su UID/GID conocido (`id <usuario>`).
- Acceso a la configuración de tu router, si vas a exponer el servicio a Internet (redirección de puertos).
- Conocimientos básicos de Linux: instalación de paquetes, edición de ficheros de texto.
- Hardware: sirve con equipo reacondicionado, una Raspberry Pi o un NAS (p. ej. Synology) — no hace falta hardware potente para una biblioteca personal.
- Puertos libres en el host: revisa la tabla en [Puertos expuestos](#puertos-expuestos) antes de levantar el stack, para evitar conflictos con otros servicios que ya tengas corriendo (por ejemplo, otra instancia de Nginx Proxy Manager).
- Una cuenta en [DuckDNS](https://www.duckdns.org/) con un subdominio reservado y su token, si vas a usar DNS dinámico.

## Stack Docker

El `docker-compose.yml` en `docker/` despliega cuatro servicios:

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

### `duckdns`
Cliente DuckDNS (`lscr.io/linuxserver/duckdns`), que actualiza periódicamente la IP pública asociada a tu subdominio `*.duckdns.org`. Necesario si tu conexión a Internet tiene IP dinámica. Requiere configurar `SUBDOMAINS` y `TOKEN` con los datos de tu cuenta en [duckdns.org](https://www.duckdns.org/).

## Puertos expuestos

Antes de levantar el stack, asegúrate de que estos puertos están libres en el host (o cámbialos en el `docker-compose.yml` si ya los usa otro servicio):

| Puerto | Servicio | Uso |
|---|---|---|
| `4533` | Navidrome | Interfaz web / API Subsonic |
| `2049` | nfs-server | NFSv4, montaje de la biblioteca desde clientes |
| `80` | npm | HTTP — necesario para el reto ACME de Let's Encrypt |
| `443` | npm | HTTPS |
| `81` | npm | Panel de administración web de Nginx Proxy Manager |
| — | duckdns | No expone puertos; solo hace peticiones salientes a la API de DuckDNS |

> Si tu servidor ya tiene otra instancia de Nginx Proxy Manager corriendo (por ejemplo en otro stack Docker), los puertos 80/443/81 entrarán en conflicto. En ese caso, reutiliza esa instancia existente en lugar de levantar `npm` de este repositorio, o remapea los puertos.

## Puesta en marcha

0. Clona el repositorio y entra en `docker/`.
1. Mueve el fichero env-example a .env 
2. Ajusta en `.env`:
   - La ruta de tu librería musical (bind mounts de `navidrome` y `nfs-server`).
   - `user: "UID:GID"` de Navidrome al usuario propietario de esa carpeta (verifica con `id <usuario>`).
   - `PERMITTED` en `nfs-server`, con la IP o rango de tu red que podrá montar el share.
   - `ND_BASEURL` con tu dominio.
   - `SUBDOMAINS` y `TOKEN` en `duckdns`, con los datos de tu cuenta en duckdns.org.
3. Levanta el stack:
   ```bash
   docker compose up -d
   ```
   Es probable que devuelva error de permisos si el uid del usuario que ejecuta docker no coincide con los permisos del directorio **data**; para resolverlo(ejemplo): 
   ```
   bash
   sudo chown apermuy data
   ```

4. Entra al panel de Nginx Proxy Manager (`http://<IP-del-host>:81`) y configura un *Proxy Host* apuntando a `navidrome:4533`, con SSL vía Let's Encrypt.
5. Accede a Navidrome desde tu dominio configurado (`tu-subdominio.duckdns.org`, o tu propio dominio si apunta a esa IP).

## Desinstalar / deshacer cambios

Para detener el stack sin perder datos (base de datos de Navidrome, configuración de NPM, certificados):

```bash
docker compose down
```

Para eliminarlo por completo, incluyendo los volúmenes con datos persistentes:

```bash
docker compose down -v
```

> `-v` borra el contenido de los volúmenes definidos por Docker, pero **no** los bind mounts (`./data`, `./npm/data`, etc.), que son carpetas normales en el host. Para borrarlos del todo, elimínalas manualmente después:
> ```bash
> rm -rf ./data ./npm
> ```
> Esto no afecta a tu biblioteca musical (`/home/apermuy/Música` o la ruta que hayas configurado), que Navidrome monta siempre en modo solo lectura.

## Clientes disponibles

Navidrome expone una API compatible con Subsonic/OpenSubsonic, así que cualquier cliente que hable ese protocolo funciona con el servidor desplegado en este repositorio. Estos son algunos de los más usados, verificados a fecha de esta revisión:

### Escritorio (Windows / macOS / Linux)

| Cliente | Plataforma(s) | Tipo | Enlace |
|---|---|---|---|
| **Feishin** | Windows, macOS, Linux | Gratis / Código abierto (GPL-3.0) | [github.com/jeffvli/feishin](https://github.com/jeffvli/feishin) |
| **Supersonic** | Windows, macOS, Linux | Gratis / Código abierto (GPL-3.0) | [github.com/supersonic-app/supersonic](https://github.com/supersonic-app/supersonic) |
| **Aonsoku** | Web app / Docker / Escritorio (Electron) | Gratis / Código abierto | [github.com/victoralvesf/aonsoku](https://github.com/victoralvesf/aonsoku) |
| **Submariner** | macOS | Gratis / Código abierto (BSD 3-clause) | [submarinerapp.com](https://submarinerapp.com/) |
| **NaviBeat** | Mac (además de móvil/TV — ver tabla siguiente) | Pago (compra única, $5.99, universal) | [navibeat.app](https://navibeat.app/) |

### Dispositivos móviles (Android / iOS)

| Cliente | Plataforma(s) | Tipo | Enlace |
|---|---|---|---|
| **Substreamer** | Android, iOS/iPadOS | Gratis / Código abierto (GPL-3.0) | [substreamer.org](https://substreamer.org/) |
| **Symfonium** | Android | Pago (con periodo de prueba) | [symfonium.app](https://symfonium.app/) |
| **Subtracks** | Android | Gratis / Código abierto (GPL-3.0, F-Droid) | [f-droid.org/packages/com.subtracks](https://f-droid.org/packages/com.subtracks/) |
| **Tempo** | Android | Gratis / Código abierto | [github.com/CappielloAntonio/tempo](https://github.com/CappielloAntonio/tempo) |
| **play:Sub** | iOS, iPadOS | Pago (compra única) | [App Store, id955329386](https://apps.apple.com/app/id955329386) |
| **Kolis Music** | iOS, iPadOS | Gratis con compras integradas | [App Store, id6757459478](https://apps.apple.com/app/id6757459478) |
| **NaviBeat** | iPhone, iPad, Apple Watch, CarPlay (además de Mac/TV) | Pago (compra única, $5.99, universal) | [navibeat.app](https://navibeat.app/) |

> NaviBeat aparece en ambas tablas: es una compra universal para todo el ecosistema Apple (iPhone, iPad, Mac, Apple TV, Apple Watch), el único caso que cruza la frontera escritorio/móvil de forma nativa.

---

*Alberto Permuy Leal — Comunidade O Zulo*

Este trabajo está licenciado bajo [Creative Commons Atribución-CompartirIgual 4.0 Internacional (CC BY-SA 4.0)](https://creativecommons.org/licenses/by-sa/4.0/deed.es).
