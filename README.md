# Producción EnAgenda

Tres sitios en el mismo servidor. Redis solo escucha en localhost. Horizon es el supervisor de las colas; no hace falta un `queue:work` aparte.

| Sitio | Dominio | Proceso |
| --- | --- | --- |
| Plataforma | `enagenda.techrep.app` | archivos estáticos de `front-plataforma/dist` |
| Panel | `adgenda.techrep.app` | Deno en `127.0.0.1:8001` |
| API | `eagapi.techrep.app` | PHP-FPM 8.3 |

Rutas en el servidor: `/var/www/enagenda/front-plataforma`, `/var/www/enagenda/admin-panel` y `/var/www/enagenda/backend-politica`.

## Redis

Copia las directivas de `redis/enagenda.conf` dentro de `/etc/redis/redis.conf`, sin duplicar `bind`, `requirepass` ni `maxmemory`. La política tiene que ser `noeviction`.

La misma clave va en `REDIS_PASSWORD` del `.env` de Laravel.

```bash
sudo systemctl enable --now redis-server
redis-cli -a 'LA_CLAVE' ping
```

## API

```bash
cp deploy/env/backend.env.example backend-politica/.env
php artisan key:generate
php artisan migrate --force
php artisan staff:create
php artisan config:cache
php artisan route:cache
```

Completa `APP_KEY`, la clave de MySQL y `REDIS_PASSWORD` en el servidor. No las dejes en el repositorio. La base `politica` tiene que existir en MySQL con `utf8mb4`. La cuenta del panel se crea con `php artisan staff:create`: pide nombre, correo y contraseña en la terminal y no las guarda en el `.env`.

`TRUSTED_PROXIES=127.0.0.1` porque Nginx termina TLS en esta máquina. Si Cloudflare llega directo a PHP, cámbialo a `cloudflare`.

## Colas

Hay dos colas en Redis, las dos las atiende Horizon:

- `default`: trabajos generales. En producción, hasta 4 procesos.
- `news`: ingesta de noticias, cada hora, con 120 segundos de margen. Hasta 2 procesos.

El programador también toma una foto de Horizon cada cinco minutos y el respaldo diario.

## Supervisor

```bash
sudo mkdir -p /var/log/enagenda
sudo cp deploy/supervisor/enagenda.conf /etc/supervisor/conf.d/enagenda.conf
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl status
```

Tras cada despliegue de la API: `php artisan horizon:terminate`. Supervisor lo vuelve a levantar.

## Nginx y Deno

El panel se construye y queda escuchando solo en local:

```bash
cd admin-panel/admin
deno task build
```

La plataforma pública hornea la URL de la API al compilar (`front-plataforma/.env.production`):

```bash
cd front-plataforma
npm ci
npm run build
```

```bash
sudo cp deploy/nginx/*.conf /etc/nginx/sites-available/
sudo ln -s /etc/nginx/sites-available/enagenda.techrep.app.conf /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/adgenda.techrep.app.conf /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/eagapi.techrep.app.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

En `eagapi.techrep.app.conf` las fotos y los videos viven en `public/media` y no hace falta `php artisan storage:link`. El bloque es `location ^~ /media/ { try_files $uri =404; }` y `client_max_body_size 52m;` para que un video de hasta 50 MB no lo corte Nginx. Después de copiar ese archivo, `sudo nginx -t && sudo systemctl reload nginx`.

Certificados: `certbot certonly --webroot -w /var/www/certbot -d enagenda.techrep.app -d adgenda.techrep.app -d eagapi.techrep.app`.

`/horizon` solo responde desde `127.0.0.1`. Desde fuera se mira con un túnel SSH. El panel del staff es `https://adgenda.techrep.app`.

Si `nginx -t` rechaza `http2 on`, el Nginx es anterior a 1.25. En cada sitio cambia las dos líneas `listen` de 443 por `listen 443 ssl http2;` y borra `http2 on;`.
