# Proxy frontal del ecosistema

Un único Caddy en Prometeus es dueño de los puertos 80 y 443 y reparte por subdominio:

| Dirección | Servicio | Alias interno |
|---|---|---|
| `mmasias.cloud-ip.cc` | Portada del ecosistema: enlaces a cada servicio ([`home/index.html`](home/index.html)) | (estático, en este Caddy) |
| `celda.mmasias.cloud-ip.cc` | CELDA ([pyCelda](https://github.com/mmasias/pyCelda)) | `pycelda-web` |
| `pulso.mmasias.cloud-ip.cc` | PULSO ([pyPULSO](https://github.com/mmasias/pyPULSO)) | `pulso-web` |

## Por qué

Antes, el Caddy de pyCelda publicaba el 80 y el 443 y era el único sitio de la máquina: cualquier aplicación nueva tenía que entrar por el repositorio y el despliegue de pyCelda. Con un frontal propio del ecosistema:

- **Cada aplicación es independiente**: su propio Compose, su propio despliegue y su propio servicio web interno. El frontal solo termina TLS y reenvía.
- **Una aplicación nueva** es un bloque de tres líneas en el [`Caddyfile`](Caddyfile). El DNS ya está resuelto con un comodín (`*.mmasias.cloud-ip.cc` CNAME `mmasias.cloud-ip.cc`, en ClouDNS): como solo la raíz es de DNS dinámico, los subdominios siguen a su IP sin actualizarse.
- **Las cookies de cada aplicación quedan en su subdominio**: una sesión de CELDA no viaja a PULSO ni al revés.

## Contrato con cada aplicación

1. Su Compose se une a la red externa `ecosistema` y su servicio web lleva un alias fijo en ella (`pycelda-web`, `pulso-web`...).
2. Ese servicio escucha HTTP en el puerto 80, **sin publicar puertos en el host**.
3. Sus cabeceras propias (CSP, `X-Frame-Options`, caché) las pone la aplicación. El frontal fija HSTS, `nosniff` y `Referrer-Policy` en todo, sustituyendo las de la aplicación si las pone (`defer`): no hace falta quitarlas de ella, ni salen duplicadas.
4. Si la aplicación necesita la IP real del cliente, confía en `X-Forwarded-For` solo desde la red `ecosistema` (rangos privados).

## Puesta en marcha (una vez, en Prometeus)

Requiere que pyCelda ya no publique el 80 ni el 443 (issue de puesta en marcha). Orden, con Manuel delante porque hay un corte de segundos:

```bash
# 0. Red compartida (una sola vez; ningún Compose la borra con `down`)
docker network create ecosistema

# 1. pyCelda con su web solo en la red interna (deja de publicar 80/443: empieza el corte)
cd ~/<pyCelda> && git pull && docker compose up -d --build

# 2. Frontal (toma 80/443 y pide los certificados: termina el corte)
cd ~/<_ECOSISTEMA_CELDA>/infra/frontal && git pull && docker compose up -d

# 3. Verificación
curl -fsS https://mmasias.cloud-ip.cc/api/health
curl -fsS https://celda.mmasias.cloud-ip.cc/api/health
docker compose logs --tail=50 frontal | grep -i -E "error|certificate obtained"
```

`pulso.` responderá `502` hasta que PULSO esté desplegado: Caddy resuelve el alias al recibir la petición, no al arrancar.

**Marcha atrás**: `docker compose down` en el frontal, revertir el commit de pyCelda y `docker compose up -d --build` en pyCelda, que vuelve a publicar el 80 y el 443.

**Cuidado en esta máquina** ([incidentes previos](https://github.com/mmasias/pyCelda/blob/main/DEPLOY.md)): no hace falta tocar firewalld (los puertos no cambian). Si aun así se hace `firewall-cmd --reload` o se reinicia Docker, comprobar que los contenedores conservan su red (`docker exec <contenedor> ip addr`) y, si no, `docker compose down && docker compose up -d`.

## Fase 2: la raíz pasa a ser la portada

Cuando el inicio de sesión de CELDA funcione por `celda.` (URI de redirección de Google OAuth y URL pública de pyCelda actualizadas), se sustituye en el `Caddyfile` el bloque de la raíz de la fase 1 por el de la fase 2, que está comentado debajo. La portada lista los servicios; cualquier otra ruta redirige a `celda.` con la misma ruta (`308`), así que los marcadores y enlaces antiguos siguen funcionando.

```bash
docker compose exec frontal caddy reload --config /etc/caddy/Caddyfile
curl -fsS -o /dev/null -w "%{http_code} %{redirect_url}\n" https://mmasias.cloud-ip.cc/guias   # 308 https://celda...
```

## Añadir un servicio

1. La aplicación cumple el contrato (red `ecosistema`, alias, puerto 80).
2. Bloque nuevo en el `Caddyfile`:

   ```
   <nombre>.mmasias.cloud-ip.cc {
   	import seguridad
   	reverse_proxy <alias>:80
   }
   ```
3. Tarjeta nueva en `home/index.html`.
4. En Prometeus: `git pull` y `docker compose exec frontal caddy reload --config /etc/caddy/Caddyfile` (sin reiniciar el contenedor ni cortar los demás servicios).

## Comprobación local

```bash
podman run --rm -v "$PWD:/etc/caddy:ro,Z" docker.io/library/caddy:2-alpine caddy validate --config /etc/caddy/Caddyfile
```
