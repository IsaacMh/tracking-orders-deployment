# Tracking Orders — despliegue con Docker

Proyecto compuesto por dos aplicaciones separadas:

- Backend Laravel en `backend-tracking-orders/`
- Frontend React + Vite + TypeScript en `frontend-tracking-orders/`

Este repositorio incluye la configuración necesaria para levantar la aplicación con contenedores Docker, con MySQL como base de datos principal, Nginx como proxy y Frontend/Backend separados.

## Arquitectura

```text
Internet
   │
   ▼
Nginx
   ├── Frontend (React + Vite)
   └── Backend (Laravel + PHP-FPM)
          │
          ▼
        MySQL
          │
          └── Redis (opcional, preparado para compatibilidad futura)
```

## Stack real detectado

### Backend
- Laravel 13
- PHP 8.3
- API REST
- autenticación basada en sesiones y tokens almacenados en base de datos
- SQLite usado localmente, migrado a MySQL para entorno Docker/producción
- notificaciones in-app, WhatsApp y push con Firebase

### Frontend
- React 19
- TypeScript
- Vite 8
- pnpm
- consume la API Laravel con `VITE_API_URL`
- usa Firebase push con service worker y `Notification` del navegador

## Requisitos locales

- Docker Engine
- Docker Compose
- Git
- una IP o dominio para producción

## Variables de entorno

Copia el archivo raíz de ejemplo a `.env`:

```bash
cp .env.example .env
```

El archivo `.env` del raíz contiene variables básicas de Docker y MySQL:

```env
APP_ENV=production
APP_DEBUG=false
APP_KEY=
APP_URL=http://localhost
APP_PORT=80

VITE_API_URL=http://localhost/api
VITE_STORAGE_URL=http://localhost/storage

MYSQL_DATABASE=tracking_orders
MYSQL_USER=tracking_user
MYSQL_PASSWORD=tracking_password
MYSQL_ROOT_PASSWORD=root_password
```

Importante:
- `APP_KEY` no debe ir hardcodeada en Git.
- Debe generarse dentro del contenedor Laravel.
- No exponer secretos reales ni contraseñas en repositorio.

## Configuración de Laravel

El backend usa un `.env.example` propio con MySQL apuntando al servicio Docker `mysql`:

```env
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=tracking_orders
DB_USERNAME=tracking_user
DB_PASSWORD=tracking_password
```

Esto está definido para Docker; nunca debe usarse `127.0.0.1` ni `localhost` dentro del contenedor backend.

## Construcción y levantamiento

Desde la raíz del proyecto:

```bash
docker compose build
docker compose up -d
```

Verificar:

```bash
docker compose ps
docker compose logs
```

Si quieres revisar solo el backend:

```bash
docker compose logs backend
```

## Generar clave de Laravel

La primera vez, genera la clave de la app:

```bash
docker compose exec backend php artisan key:generate
```

## Ejecutar migraciones y seeders

```bash
docker compose exec backend php artisan migrate
docker compose exec backend php artisan db:seed
```

## Acceso a contenedores

```bash
docker compose exec backend bash
docker compose exec mysql mysql -u root -p
```

## URLs esperadas

Cuando los contenedores estén activos, la app quedará disponible en:

- Frontend: http://localhost/
- API Laravel: http://localhost/api
- Laravel direct: http://localhost/api/usuario/login

## Notificaciones y servicios externos

La aplicación usa:

- Firebase para push del navegador
- Meta WhatsApp API para mensajes
- base de datos para notificaciones in-app
- no usa WebSockets ni Reverb como parte del flujo principal

Por eso no se añadió un servicio de WebSockets en Docker, porque la funcionalidad actual no depende de esa tecnología para funcionar correctamente.

## Producción

Para desplegar en un VPS o servidor de producción, se recomienda:

- Ubuntu 22.04 LTS o compatible
- Docker Engine + Docker Compose
- dominio apuntando a la IP pública
- HTTPS con Caddy, Traefik o Nginx + Certbot
- backups de MySQL
- `.env` real guardado en el servidor, no en Git
- `APP_ENV=production`
- `APP_DEBUG=false`

## SQLite → MySQL

1. Mantén la SQLite original mientras validas MySQL.
2. Levanta el servicio MySQL con Docker.
3. Ejecuta migraciones en el backend.
4. Revisa que la app responda correctamente.
5. Luego, si lo necesitas, importa los datos relevantes a MySQL.

Ejemplo de migración:

```bash
docker compose exec backend php artisan migrate
```

Si ya tienes datos importantes en SQLite, conviértelos con un proceso seguro en un entorno de prueba antes de eliminar la base original.

## Comandos de parada y limpieza

```bash
docker compose down
docker compose down -v
```

> `-v` elimina volúmenes persistentes de Docker, útil para resetear completamente la base de datos local.

## Detección de fallos rápida

```bash
docker compose ps
docker compose logs backend
docker compose logs mysql
docker compose logs nginx
```

Si Laravel no arranca:

```bash
docker compose exec backend php artisan config:clear
docker compose exec backend php artisan cache:clear
docker compose exec backend php artisan migrate
```

## Archivos importantes del proyecto

- [docker-compose.yml](docker-compose.yml)
- [backend-tracking-orders/Dockerfile](backend-tracking-orders/Dockerfile)
- [frontend-tracking-orders/Dockerfile](frontend-tracking-orders/Dockerfile)
- [backend-tracking-orders/.env.example](backend-tracking-orders/.env.example)
- [frontend-tracking-orders/.env.example](frontend-tracking-orders/.env.example)
- [nginx/default.conf](nginx/default.conf)

## Nota final

Este despliegue está preparado para desarrollo y posterior puesta en producción. Para un entorno real, solo faltan los valores reales de:

- `APP_KEY`
- Firebase Cloud Messaging
- Meta WhatsApp
- credenciales reales de la base de datos

No se deben hardcodear en el repositorio.
