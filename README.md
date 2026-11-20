# Proyecto final · Corte 3 · Sistemas Operativos 103027

Grupo **sokj** · Juan Camilo Orozco Londoño y Karolay <Apellido> · UAM

Una página web en AWS que muestra una tabla de sistemas operativos. Los datos los trae de una API con base de datos que corre en Azure con Docker. Cada parte tiene su subdominio y HTTPS.

| Parte | Servidor | URL | Qué corre |
|---|---|---|---|
| 1 · Frontend | AWS EC2 t3.micro, Ubuntu 24.04 | https://sokj.scaleautomatization.tech | Nginx instalado a mano, sin Docker, sirviendo `index.html` y `config.js` |
| 2 · Backend | Azure VM Standard_B2ats_v2, Ubuntu 24.04, Chile Central | https://api-sokj.scaleautomatization.tech | Nginx + certbot como proxy inverso hacia dos contenedores con Docker Compose: `backend` (Flask + gunicorn) y `db` (PostgreSQL 17) |

Imagen del backend: https://hub.docker.com/r/jucalll/so-backend (etiqueta `1.0`, linux/amd64)

## Cómo se conectan

1. El navegador descarga la página desde EC2.
2. El JavaScript de la página lee `API_URL` de `config.js` y pide `/api/sistemas` a la API de Azure. La petición la hace el navegador, no el servidor de EC2.
3. En Azure, Nginx recibe la petición en el 443 y la pasa a `127.0.0.1:8000`, donde escucha el contenedor `backend`.
4. El backend consulta a Postgres por la red interna de Compose, usando el nombre del servicio (`db`) como host.
5. La API responde con el encabezado `Access-Control-Allow-Origin: https://sokj.scaleautomatization.tech` (CORS), y el navegador deja que la página use los datos.

La base de datos no publica ningún puerto, y el backend solo se publica en `127.0.0.1`. Desde internet solo están abiertos el 22, el 80 y el 443.

## Archivos que agregamos

| Archivo | Para qué |
|---|---|
| `backend/Dockerfile` | Imagen del backend: `python:3.12-slim`, dependencias y gunicorn en `0.0.0.0:8000` |
| `backend/.dockerignore` | Evita que el `.env` y la caché de Python entren a la imagen |
| `compose.yaml` | Servicios `db` y `backend`, volumen `datos-db`, healthcheck y reinicio automático |
| `nginx/aws-sokj.conf` | Sitio de la página en EC2, con HTTPS |
| `nginx/azure-api-sokj.conf` | Proxy inverso de la API en Azure, con HTTPS |
| `frontend/config.js` | Apunta a `https://api-sokj.scaleautomatization.tech` |
| `README-codigo-base.md` | README original del código base (prueba en local) |

El `.env` real no está en el repositorio. La plantilla es `backend/.env.example`.

## Parte 1 · AWS (Karolay)

1. Instancia EC2 t3.micro con Ubuntu 24.04, llave `so-final-aws`, Security Group con 22, 80 y 443, e IP elástica.
2. En la instancia:

```bash
sudo apt update
sudo apt install -y nginx git certbot python3-certbot-nginx
git clone https://github.com/JuCalll/so-final.git
sudo mkdir -p /var/www/sokj
sudo cp -r ~/so-final/frontend/* /var/www/sokj/
```

3. Sitio en `/etc/nginx/sites-available/sokj` (versión final en `nginx/aws-sokj.conf`):

```bash
sudo ln -s /etc/nginx/sites-available/sokj /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
sudo certbot --nginx -d sokj.scaleautomatization.tech
```

## Parte 2 · Azure (Juan)

### Imagen (en el PC, no en la VM)

```bash
cd backend
docker build -t jucalll/so-backend:1.0 .
docker run --rm -p 8000:8000 -e DATABASE_URL=postgresql://x:x@localhost:5432/x jucalll/so-backend:1.0
docker push jucalll/so-backend:1.0
```

El `DATABASE_URL` de prueba es necesario porque `app.py` lo lee al arrancar.

### Máquina virtual

- Azure for Students solo permite cinco regiones (canadacentral, spaincentral, brazilsouth, chilecentral, westus), y B1s sale `NotAvailableForSubscription`. Por eso usamos Chile Central con Standard_B2ats_v2 (2 vCPU, 1 GiB, x86).
- NSG con 22, 80 y 443. IP pública estática: 68.211.88.154.
- 2 GB de swap, porque la VM tiene 1 GiB de RAM.
- Docker con `curl -fsSL https://get.docker.com | sudo sh` y el usuario en el grupo `docker`.

### Compose

```bash
git clone https://github.com/JuCalll/so-final.git
cd so-final
cp backend/.env.example .env   # HOST_DE_LA_DB pasa a ser db y se pone una clave propia
docker compose up -d
docker compose ps
curl http://127.0.0.1:8000/api/sistemas
```

- `db` usa `postgres:17-alpine`, guarda los datos en el volumen `datos-db` y ejecuta `db/init.sql` la primera vez, con el volumen vacío.
- `backend` espera a que `db` pase el healthcheck (`pg_isready`) y se publica solo en `127.0.0.1:8000`.
- Los dos tienen `restart: unless-stopped`: probamos reiniciar la VM y volvieron a levantar solos.

### Nginx y HTTPS

```bash
sudo apt install -y nginx certbot python3-certbot-nginx
# sitio /etc/nginx/sites-available/api-sokj con proxy_pass http://127.0.0.1:8000
sudo ln -s /etc/nginx/sites-available/api-sokj /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
sudo certbot --nginx -d api-sokj.scaleautomatization.tech
```

## Integración

- `frontend/config.js`: `const API_URL = "https://api-sokj.scaleautomatization.tech";`
- `.env` de Azure: `ORIGEN_PERMITIDO=https://sokj.scaleautomatization.tech`

## Verificación

```bash
curl https://api-sokj.scaleautomatization.tech/api/salud      # {"estado":"ok"}
curl https://api-sokj.scaleautomatization.tech/api/sistemas   # 8 sistemas operativos
curl -s -I -H "Origin: https://sokj.scaleautomatization.tech" https://api-sokj.scaleautomatization.tech/api/sistemas | grep -i access-control
```

Al abrir https://sokj.scaleautomatization.tech, la tabla muestra los 8 sistemas, de UNIX (1969) a Android (2008).
