# Poli-REDI: operación del stack

Corte: 22 de septiembre de 2026. Evidencia: resultados de ejecución en Debian
compartidos por el operador. Este documento describe el entorno instalado y
sustituye el estado de propuesta del README inicial.

## Configuración instalada

- Repositorio operativo: `/home/admin/projects/poliredi-infra`.
- Compose: `compose/compose.yaml`; Caddy: `compose/Caddyfile`.
- Podman 5.4.2; proveedor `/usr/bin/podman-compose`, versión 1.3.0.
- Unidad: `~/.config/systemd/user/poliredi-stack.service`, habilitada; Linger=yes.
- PostgreSQL 16: `poliredi_postgres_1`, volumen externo
  `poliredi-postgres-mvp1-data`, puerto local 55432.
- API: `poliredi_backend_1`, imagen `localhost/poliredi-backend:mvp2`.
- Web: `poliredi_web_1`, Caddy 2.11.4, puerto local 8443.
- Frontend compilado: `/home/admin/projects/poliredi/frontend/dist`.
- Funnel: `https://desktop-epot7cf.tail16d8fb.ts.net` hacia
  `https+insecure://localhost:8443`.
- Caddy admite localhost y el dominio de Funnel. Conservar el Caddyfile de
  Debian; la primera plantilla Windows solo admitía localhost.

No hay migración de datos ni cambio de versión mayor PostgreSQL en este corte.
Los Quadlets anteriores están retirados del directorio activo. La definición
PostgreSQL 17 del repositorio antiguo no corresponde al volumen vigente.

## Variables y secretos

Los archivos `~/.config/poli-redi/compose.env`, `backend.env` y
`~/.config/containers/systemd/poliredi-postgres.env` permanecen fuera de Git.
Compose fuerza DATABASE_URL vacía y PGHOST=postgres para usar PG*.
La contraseña se inyecta desde el secreto externo
`poliredi-postgres-app-password` como PGPASSWORD, usando `type: env`, compatible
con este proveedor Podman Compose. No es una configuración portable sin revisión
a Docker Compose.

El secreto antiguo fallaba por la red. Se validó la contraseña de DATABASE_URL
por la red de Compose, se reemplazó el secreto y se recreó el backend. No se
alteró la contraseña del rol en PostgreSQL. La prueba inicial por loopback no
demostraba autenticación por contraseña debido a las reglas locales de acceso.
Se conservaron los demás secretos, incluido `poliredi-join-code-keys`; su sola
existencia no acredita que la imagen actual lo consuma.

## Operación diaria

```bash
systemctl --user status poliredi-stack.service --no-pager
podman ps --format 'table {{.Names}}\t{{.Status}}'
curl --max-time 10 --fail --insecure https://localhost:8443/api/health
```

Para iniciar o detener toda la aplicación (la parada interrumpe el servicio):

```bash
systemctl --user start poliredi-stack.service
systemctl --user stop poliredi-stack.service
```

Registros:

```bash
export PODMAN_COMPOSE_PROVIDER=/usr/bin/podman-compose
cd ~/projects/poliredi-infra/compose
podman compose --env-file ~/.config/poli-redi/compose.env logs --tail=80
```

La unidad oneshot queda `active (exited)` al finalizar Compose: no monitoriza
continuamente la salud de la aplicación. Revisar también contenedores y API.
Durante el arranque Caddy puede devolver 502 mientras PostgreSQL alcanza healthy
y arranca la API. No se interpreta como fallo persistente si después se recupera.
El proveedor 1.3.0 mostró errores de nombres ya ocupados al ejecutar up sobre
contenedores existentes; terminó con código 0 y los tres quedaron saludables.
Conservar esta incidencia: no ocultar los errores ni asumir salud por el código 0.

## Evidencia de validación

- Configuración Compose y Caddy aceptadas; wget y ejecutable backend presentes.
- Tres contenedores healthy; API local mediante Caddy devuelve status=ok.
- Host público probado contra Caddy local: HTTP 200.
- Usuario confirmó funcionamiento desde navegador y de los flujos consultados.
  No equivale a una suite exhaustiva ni demuestra acceso desde red móvil separada.
- Reinicio de Debian con `wsl --terminate Debian` y posterior apertura:
  systemd inició Compose sin comando manual; tras dos 502 la API devolvió ok.
- Servicio habilitado y Linger=yes. Esto inicia el stack cuando Debian arranca;
  no programa el inicio de WSL al encender Windows ni verifica operación desatendida.
- curl al dominio público desde Debian agotó el tiempo TLS. Tailscale estaba
  conectado y netcheck tenía IPv4/UDP; el navegador funcionó. Causa pendiente.
- Respaldo lógico creado; pg_restore --list pudo leer su catálogo. Restauración
  completa aún no ensayada.

## Respaldos y recuperación

Respaldo anterior al corte:
`/home/admin/projects/poliredi-backups/pre-compose-20260922-172926`.
Contiene dump, catálogo, configuración privada y copia de Quadlets.
Quadlets retirados:
`/home/admin/projects/poliredi-backups/corte-compose-20260922-173956/quadlet`.
Son ubicaciones privadas, no se deben añadir sus contenidos a Git.

Para recuperar ante un problema de aplicación, preferir restaurar la imagen o
configuración validada y recrear solo el servicio afectado. No restaurar la base
ni borrar el volumen para resolver errores de red o credenciales.

Para abandonar Compose: detener y deshabilitar poliredi-stack.service, ejecutar
Compose down SIN -v y confirmar que ningún contenedor usa el volumen antes de
iniciar otro PostgreSQL. Conservar el volumen y los secretos.

Los Quadlets previos NO son un rollback funcional listo: la API buscaba
poliredi-postgres sin que la base MVP1 estuviera en su red. Antes de reactivarlos,
corregir red/alias para usar la base PostgreSQL 16 y el secreto validado. Revisar
también Caddy y Funnel. Restaurar los archivos por sí solo reproduce el fallo
anterior. Este retorno no se ha ensayado.

Pendientes: ensayo de restauración en base aislada, prueba pública desde otra
red, investigación del timeout desde Debian y, si se requiere, inicio de WSL
con Windows. No se eliminaron contenedores antiguos detenidos.
