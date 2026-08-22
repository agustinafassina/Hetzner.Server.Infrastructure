# Hetzner.Server.Infrastructure
Infra del VPS (Hetzner + Ubuntu + Docker). **Un solo Compose** levanta Caddy y los contenedores. Cada app vive en su propio repo (código + `Dockerfile`); este repo solo orquesta.

```text
Internet → Cloudflare → Caddy (80/443) → servicio:puerto
```

No publiques puertos de las apps a Internet. Solo Caddy escucha 80 y 443.

Un push a `main` de **este** repo (o *Run workflow*) hace `git pull` de infra y de cada app, y `docker compose up -d --build`. Las variables viven en el `.env` **del servidor**. No se commitean.

## Layout en el VPS
```text
/opt/infra         ← este repo + .env
/opt/my-project    ← Librarian's Challenge (game)
/opt/wishes-app    ← Wishes.App
/opt/wishes-data   ← JSON de usuarios (sobrevive rebuilds y compose down)
```

Hoy el Compose incluye `game` y `wishes`. La API va comentada hasta que la sumes.

## Primera vez en el server
Como `root` (o el usuario que use Docker):

```bash
mkdir -p /opt/infra /opt/my-project /opt/wishes-app /opt/wishes-data
```

Cloná **este** repo en `/opt/infra` y cada app en su carpeta (misma deploy key de GitHub si son privados).

Si Caddy está instalado en el host, paralo para no chocar el puerto 80:

```bash
systemctl stop caddy
systemctl disable caddy
```

```bash
cd /opt/infra
cp .env.example .env
nano .env
```

Mínimo:

```env
GAME_PATH=/opt/my-project
GAME_SITE=http://localhost
WISHES_PATH=/opt/wishes-app
WISHES_DATA_PATH=/opt/wishes-data
WISHES_SITE=wishes.localhost
WISHES_APP_BASE_URL=http://wishes.localhost
```

Wishes (Auth0), en el mismo `.env`:

- `WISHES_AUTH0_DOMAIN` = campo **Domain** de la app en el dashboard de Auth0 (`algo.us.auth0.com`), **sin** `https://`
- `WISHES_AUTH0_CLIENT_ID` / `CLIENT_SECRET` / `SECRET` (`SECRET` ≥ 32 chars)
- `WISHES_AUTH0_AUDIENCE` opcional: Identifier de la API, o vacío
- `WISHES_APP_BASE_URL` = URL pública **sin** barra final

Esas vars entran en el **build** de wishes. Si las cambiás, hay que rebuildear: `docker compose up -d --build wishes`.

En Auth0 (producción): Callback `https://tudominio.com/auth/callback`, Logout y Web Origins con la misma URL.

Con dominio (Cloudflare A/AAAA a la IP del VPS, SSL **Full**):

```env
GAME_SITE=game.tudominio.com
WISHES_SITE=wishes.tudominio.com
WISHES_APP_BASE_URL=https://wishes.tudominio.com
CADDY_EMAIL=tu@email.com
```

Levantar:

```bash
cd /opt/infra
docker compose up -d --build
docker compose ps
curl -f http://127.0.0.1/health
```

Tiene que responder `ok` (Caddy → `game`). Wishes: el hostname de `WISHES_SITE`.

## Desarrollo en la PC
Con este repo al lado de los clones:

```bash
cp .env.example .env
```

En `.env`, paths locales, por ejemplo:

```env
GAME_PATH=../Library.LibrarianChallenge.Game
WISHES_PATH=../Wishes.App
WISHES_DATA_PATH=./data/wishes
GAME_SITE=http://localhost
WISHES_SITE=wishes.localhost
```

```bash
docker compose up -d --build
```

Abrí <http://localhost> (game) y <http://wishes.localhost> (wishes). Si 80 está ocupado, cambiá el mapeo en `compose.yml` o liberá el puerto.

## Cómo agregar otro contenedor
1. Cloná el repo en `/opt/...`
2. Servicio en `compose.yml` (`build.context` = ruta del clone)
3. Bloque en `Caddyfile` (`reverse_proxy nombre:puerto`)
4. `*_PATH` y `*_SITE` en `.env`
5. `docker compose up -d --build`

Hay un ejemplo comentado para la API. No metas el código de las apps en este repo.

## GitHub Actions (este repo)
**Secrets** (no van en el `.env`):

| Secreto | Valor |
|---------|--------|
| `SERVER_HOST` | IP del VPS |
| `SERVER_USER` | `root` o `deploy` (el user con el que `ssh` entra) |
| `SERVER_SSH_KEY` | clave **privada** para entrar al VPS |
| `SERVER_PORT` | opcional, `22` |
| `SERVER_SSH_KNOWN_HOSTS` | opcional, `ssh-keyscan` |

**Variables:**

| Variable | Ejemplo |
|----------|---------|
| `INFRA_PATH` | `/opt/infra` |
| `GAME_PATH` | `/opt/my-project` |
| `WISHES_PATH` | `/opt/wishes-app` |
| `HEALTHCHECK_URL` | `https://game.tudominio.com/health` |

El `.env` del server **no** se pisa en el deploy (`git reset` no lo toca). Auth0 y el resto de vars de apps van ahí, no en Secrets.

## Comandos útiles
```bash
cd /opt/infra
docker compose ps
docker compose logs -f caddy
docker compose logs -f game
docker compose logs -f wishes

# actualizar código de una app y rebuild
git -C /opt/my-project fetch origin && git -C /opt/my-project reset --hard origin/main
docker compose up -d --build --force-recreate game

git -C /opt/wishes-app fetch origin && git -C /opt/wishes-app reset --hard origin/main
docker compose up -d --build --force-recreate wishes

# si Docker reusa la imagen vieja
docker compose build --no-cache game
docker compose up -d --force-recreate game

docker compose down                  # apaga contenedores; no borra /opt/wishes-data
# no uses: docker compose down -v   # borra certificados de Caddy
```
