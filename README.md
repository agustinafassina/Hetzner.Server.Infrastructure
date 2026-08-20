# Hetzner.Server.Infrastructure
Infra del VPS (Hetzner + Ubuntu + Docker). **Un solo Compose** levanta Caddy y los contenedores. Cada app vive en su propio repo (código + `Dockerfile`); este repo solo orquesta.

```text
Internet → Cloudflare → Caddy (80/443) → servicio:puerto
                                      → servicio:puerto
```

No publiques puertos de las apps a Internet. Solo Caddy escucha 80 y 443.

## Idea
| Pieza | Dónde |
|-------|--------|
| Este repo | Caddy + `compose.yml` + `.env` |
| Cada app | Repo aparte, clonado en `/opt/...`, un servicio en Compose |

Un push a `main` **de este repo** (o *Run workflow*) hace:

1. `git pull` de infra  
2. `git pull` de cada repo de app configurado  
3. `docker compose up -d --build`  

Las variables de cada contenedor viven en el `.env` **de este repo en el servidor**. No se commitean.

## Layout en el VPS
```text
/opt/infra         ← clone de este repo + .env
/opt/my-project    ← Librarian's Challenge (game)
/opt/wishes-app    ← Wishes.App
/opt/wishes-data   ← JSON de usuarios (volumen; sobrevive rebuilds)
```

Hoy el Compose incluye `game` y `wishes`. La API va comentada hasta que la sumes. Cada app es un contenedor: hostname → `reverse_proxy`.

## Primera vez en el server
Como `root`, usuario `deploy` (o el que uses) con Docker:

```bash
mkdir -p /opt/infra /opt/my-project /opt/wishes-app /opt/wishes-data
chown deploy:deploy /opt/infra /opt/my-project /opt/wishes-app /opt/wishes-data
```

Como `deploy`, cloná **este** repo en `/opt/infra` y cada app en su carpeta (misma deploy key de GitHub).

Si tenés Caddy instalado en el host (`systemctl status caddy`), **paralo** para no chocar el puerto 80:

```bash
systemctl stop caddy
systemctl disable caddy
```

```bash
cd /opt/infra
cp .env.example .env
nano .env
```

Valores mínimos (`GAME_PATH` y `WISHES_PATH` apuntan a cada clone):

```env
GAME_PATH=/opt/my-project
GAME_SITE=http://localhost
WISHES_PATH=/opt/wishes-app
WISHES_DATA_PATH=/opt/wishes-data
WISHES_SITE=wishes.localhost
WISHES_APP_BASE_URL=http://wishes.localhost
```

Wishes necesita Auth0 en el `.env` (`WISHES_AUTH0_*`, incluido `WISHES_AUTH0_AUDIENCE`, y `WISHES_AUTH0_SECRET` ≥ 32 chars). `WISHES_APP_BASE_URL` es la URL pública **sin** barra final.

Cuando tengas dominio (Cloudflare A/AAAA a la IP del VPS, SSL Full):

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

Tiene que responder `ok` (Caddy → contenedor `game`).

## Desarrollo en la PC
Con este repo al lado de los clones de las apps:

```bash
cp .env.example .env
# GAME_PATH=../Library.LibrarianChallenge.Game
# WISHES_PATH=../Wishes.App
# GAME_SITE=http://localhost
# WISHES_SITE=wishes.localhost
docker compose up -d --build
```

Abrí <http://localhost> (game) y <http://wishes.localhost> (wishes). Si 80 está ocupado, cambiá el mapeo en `compose.yml` o liberá el puerto.

## Cómo agregar otro contenedor
1. Cloná el repo de la app en `/opt/...`  
2. Agregá el servicio en `compose.yml` (`build.context` = ruta del clone).  
3. Agregá un bloque en `Caddyfile` (`reverse_proxy nombre:puerto`).  
4. Definí `*_PATH` y `*_SITE` en `.env`.  
5. `docker compose up -d --build`  

Hay un ejemplo comentado para la API. No metas el código de las apps en este repo.

## GitHub Actions (este repo)
Secretos:

| Secreto | Valor |
|---------|--------|
| `SERVER_HOST` | IP del VPS |
| `SERVER_USER` | `deploy` |
| `SERVER_SSH_KEY` | clave privada para entrar por SSH |
| `SERVER_PORT` | opcional, `22` |
| `SERVER_SSH_KNOWN_HOSTS` | opcional, `ssh-keyscan` |

Variables:

| Variable | Ejemplo |
|----------|---------|
| `INFRA_PATH` | `/opt/infra` |
| `GAME_PATH` | `/opt/my-project` |
| `WISHES_PATH` | `/opt/wishes-app` |
| `HEALTHCHECK_URL` | `https://game.tudominio.com/health` |

El `.env` del server **no** se pisa en el deploy (`git reset` no lo toca: está en `.gitignore`).

## Comandos útiles
```bash
cd /opt/infra
docker compose ps
docker compose logs -f caddy
docker compose logs -f game
docker compose logs -f wishes
docker compose up -d --build game    # reconstruir un servicio
docker compose up -d --build wishes
docker compose down                  # apaga todo el stack
```

#### Version
1.0