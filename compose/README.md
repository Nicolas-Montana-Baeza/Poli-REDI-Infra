# Stack único con Podman Compose

Preparado el 22-09-2026. Estado: propuesta de adopción, aún no desplegada.

## Evidencia del entorno Debian

La base activa es `poliredi-postgres-mvp1`, PostgreSQL 16, con el volumen
`poliredi-postgres-mvp1-data`. El repositorio `~/projects/poliredi-infra` tiene
cambios locales y una definición PostgreSQL 17 que NO representa esa base.
No usar esa imagen 17 con el volumen 16. Solo PostgreSQL estaba ejecutándose
en la inspección. Compose seleccionaba un ejecutable Windows de Docker que falla.

Este paquete reúne postgres, backend y web. Conserva el volumen como externo,
sin inicialización ni migraciones automáticas. Está destinado a adoptar la base
existente, no a aprovisionar una base vacía. El backend espera el healthcheck de
PostgreSQL y se reinicia si falla; esto no implementa reintentos en el código Go.
Caddy puede arrancar antes del backend y volver a conectar cuando esté disponible.

## Preparación en Debian

Instalar el proveedor Linux y fijarlo para evitar el ejecutable Windows:

```bash
sudo apt update
sudo apt install podman-compose
export PODMAN_COMPOSE_PROVIDER=/usr/bin/podman-compose
podman compose version
```

Copiar esta carpeta al repositorio de infraestructura como `compose/` tras
revisar sus cambios locales. Mantener el código de aplicación en
`~/projects/poliredi`; esta configuración consume imágenes ya construidas y un
frontend ya compilado. Copiar `.env.example` a
`~/.config/poli-redi/compose.env` y ajustar rutas e imagen.

Usar `~/.config/poli-redi/backend.env` con permisos 0600 y con:

- La contraseña se inyecta desde el secreto externo Podman
  `poliredi-postgres-app-password` como `PGPASSWORD`; no copiarla al archivo.
  La sintaxis `type: env` se comprobó en podman-compose 1.3.0 y no es portable
  automáticamente a Docker Compose.
- `ENTRA_TENANT_ID`, `ENTRA_API_CLIENT_ID`, `ENTRA_ISSUER` vigentes.
- `CORS_ALLOWED_ORIGINS`: URL pública y orígenes locales usados realmente.
- Configuración vigente de códigos de invitación, incluidas claves/versiones.

Conservar todas las claves actuales: regenerarlas puede invalidar invitaciones.
No copiar una URL con host localhost: Compose fuerza PGHOST=postgres y vacía
DATABASE_URL para usar PG*. Las credenciales del owner se leen del archivo
existente de Quadlet y no cambian la contraseña de una base ya inicializada.

Compilar el frontend con `VITE_API_BASE_URL=/api`, `VITE_MVP_SCOPE=mvp2`,
`VITE_DEV_AUTH_ENABLED=false` y las URLs/scopes reales de Entra. Verificar que la
imagen backend elegida corresponde al código validado y contiene wget (la
imagen Alpine construida con el Containerfile del proyecto lo incluye).

Validar configuración localmente; la salida puede contener secretos, no compartirla:

```bash
cd ~/projects/poliredi-infra/compose
podman compose --env-file ~/.config/poli-redi/compose.env config >/dev/null
```

## Corte controlado (pendiente de ejecutar)

1. Guardar respaldo lógico verificable de la base y una copia de los Quadlets,
   archivos privados y configuración Funnel. Detener escrituras durante el corte.
2. Detener los servicios de usuario `poliredi-web`, `poliredi-api` y
   `poliredi-postgres`. Retirar sus archivos `.container` del directorio activo
   de Quadlet hacia un directorio de respaldo; ejecutar daemon-reload. No basta
   con detenerlos: WantedBy puede volver a iniciarlos en otra sesión.
3. Confirmar que no queda ningún proceso PostgreSQL usando el volumen ni un
   servicio ocupando 55432 o 8443. Nunca iniciar dos servidores sobre ese volumen.
4. Arrancar desde esta carpeta:

```bash
export PODMAN_COMPOSE_PROVIDER=/usr/bin/podman-compose
podman compose --env-file ~/.config/poli-redi/compose.env up -d
podman compose --env-file ~/.config/poli-redi/compose.env ps
curl --fail --insecure https://localhost:8443/api/health
```

5. Validar login, /api/me, disponibilidad, reserva, código de invitación y datos
   previos. El health HTTP no sustituye una consulta autenticada a la base.
6. Revisar Funnel: destino HTTPS local 8443, compatible con certificado interno.
   El Caddyfile está preparado para backend localhost; verificar el Host enviado
   por el proxy y adaptar al dominio público si conserva otro Host.

Caddy publica solo en loopback. Los volúmenes Caddy nuevos generan certificado
interno nuevo; se debe revalidar la confianza/configuración TLS de Funnel.
No se modificó Funnel automáticamente.

Operación desde la carpeta compose:

```bash
podman compose --env-file ~/.config/poli-redi/compose.env logs --tail=100
podman compose --env-file ~/.config/poli-redi/compose.env stop
podman compose --env-file ~/.config/poli-redi/compose.env up -d
```

## Reinicio del equipo y rollback

`restart: unless-stopped` gestiona reinicios de contenedores; probar aparte el
arranque tras reiniciar WSL. Antes de declarar el despliegue completo, configurar
un único servicio systemd de usuario que ejecute Compose (con proveedor Linux,
directorio y env-file absolutos), habilitar linger si se necesita sin sesión y
comprobarlo. No reactivar los tres Quadlets a la vez.

Para rollback: detener y retirar el stack con `down` sin `-v`, confirmar que no
usa el volumen, restaurar los Quadlets originales y hacer daemon-reload; arrancar
PostgreSQL, API y web. Restaurar configuración TLS/Funnel previa si se cambió.
No borrar volúmenes. Este cambio no altera el esquema SQL.

## Validación pendiente

- Parsear Compose con el proveedor nativo instalado.
- Arranque de los tres servicios, autenticación y persistencia.
- Funnel y reinicio WSL.
- Backup/restauración y rollback.

Referencias: https://docs.podman.io/en/latest/markdown/podman-compose.1.html
y https://docs.docker.com/reference/compose-file/volumes/.
