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
/opt/infra     ← clone de este repo + .env
/opt/app-a     ← clone de una app (ejemplo: el juego)
/opt/app-b     ← clone de otra app (ejemplo: la API)
```

Hoy el Compose incluye el servicio `game` (Librarian's Challenge). La API va comentada hasta que la sumes. Ambos son contenedores iguales de cara a Caddy: hostname → `reverse_proxy`.

## Primera vez en el server
Como `root`, usuario `deploy` (o el que uses) con Docker:

```bash
mkdir -p /opt/infra /opt/my-project
chown deploy:deploy /opt/infra /opt/my-project
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

Valores mínimos (el juego es un servicio más; `GAME_PATH` apunta a su clone):

```env
GAME_PATH=/opt/my-project
GAME_SITE=http://localhost
```

Cuando tengas dominio (Cloudflare A/AAAA a la IP del VPS, SSL Full):

```env
GAME_SITE=game.tudominio.com
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
# GAME_SITE=http://localhost
docker compose up -d --build
```

Abrí <http://localhost> (puerto 80). Si 80 está ocupado, cambiá el mapeo en `compose.yml` o liberá el puerto.

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
| `HEALTHCHECK_URL` | `https://game.tudominio.com/health` |

El `.env` del server **no** se pisa en el deploy (`git reset` no lo toca: está en `.gitignore`).

## Comandos útiles
```bash
cd /opt/infra
docker compose ps
docker compose logs -f caddy
docker compose logs -f game
docker compose up -d --build game    # reconstruir un servicio
docker compose down                  # apaga todo el stack
```
