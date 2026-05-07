# EV2 Frontend - Innovatech Chile

Frontend React + Vite para la aplicación de despachos de Innovatech Chile. Esta aplicación está completamente containerizada y preparada para CI/CD en AWS.

## 📋 Tabla de Contenidos

- [Requisitos Previos](#requisitos-previos)
- [Instalación Local](#instalación-local)
- [Ejecución con Docker](#ejecución-con-docker)
- [Variables de Entorno](#variables-de-entorno)
- [Estructura del Proyecto](#estructura-del-proyecto)
- [Pipeline CI/CD](#pipeline-cicd)
- [Despliegue en EC2](#despliegue-en-ec2)
- [Troubleshooting](#troubleshooting)

---

## 🔧 Requisitos Previos

- **Node.js** 18+ (para desarrollo local)
- **Docker** 20.10+ (para containerización)
- **Docker Compose** 1.29+ (para orquestación)
- **Git** 2.25+ (para control de versiones)
- **AWS CLI** v2 (para operaciones en ECR)
- Credenciales AWS (access key + secret key)
- Cuenta Docker Hub (para publicar imágenes)

---

## 💻 Instalación Local

### 1. Clonar el repositorio

```bash
git clone https://github.com/FabianReyes02/EV2front.git
cd EV2front
```

### 2. Instalar dependencias

```bash
npm install
```

### 3. Configurar variables de entorno

```bash
cp .env.example .env
```

Edita `.env` con tus valores:

```env
VITE_API_URL=http://localhost:8080  # URL del Backend
FRONTEND_PORT=3000                   # Puerto del Frontend
NODE_ENV=development                 # Ambiente
```

### 4. Ejecutar en desarrollo

```bash
npm run dev
```

La aplicación estará disponible en `http://localhost:5173`

### 5. Compilar para producción

```bash
npm run build
npm run preview
```

---

## 🐳 Ejecución con Docker

### Compilar imagen localmente

```bash
docker build -t ev2-frontend:latest .
```

**Explicación del Dockerfile:**
- **Stage 1 (Builder):** Compila React con Vite usando Node 20 Alpine
  - Instala dependencias
  - Ejecuta `npm run build`
  - Genera carpeta `/dist`
  
- **Stage 2 (Production):** Sirve build con Nginx Alpine
  - Copia solo `/dist` (imagen más ligera)
  - Configura usuario no-root (`nginx`)
  - Expose puerto 80

### Ejecutar con docker-compose

```bash
# Desarrollo/Testing
docker-compose up

# En background
docker-compose up -d

# Ver logs
docker-compose logs -f frontend

# Detener
docker-compose down
```

**Acceso:** http://localhost:3000

### Ejecutar contenedor directamente

```bash
docker run -d \
  --name ev2-frontend \
  -p 3000:80 \
  -e VITE_API_URL=http://localhost:8080 \
  -e NODE_ENV=production \
  ev2-frontend:latest
```

---

## 🌐 Variables de Entorno

### Desarrollo Local

**Archivo:** `.env`

```env
VITE_API_URL=http://localhost:8080  # Backend en local
FRONTEND_PORT=3000
NODE_ENV=development
```

### Docker/Producción

Se definen en `docker-compose.yml` o en el comando `docker run`:

```env
VITE_API_URL=http://backend-api:8080  # O IP pública de Backend
NODE_ENV=production
```

### En EC2 (Post-deployment)

El pipeline CI/CD usa GitHub Secrets:

```env
BACKEND_API_URL=<secret>  # URL producción del backend
```

---

## 📁 Estructura del Proyecto

```
EV2front/
├── .github/
│   └── workflows/
│       └── deploy.yml              # Pipeline CI/CD
├── public/                         # Archivos estáticos
├── src/
│   ├── componentes/               # Componentes React reutilizables
│   ├── Routes/                    # Páginas/rutas principales
│   ├── assets/                    # Imágenes, fonts, etc
│   ├── index.css                  # Estilos globales (TailwindCSS)
│   └── main.jsx                   # Entry point
├── Dockerfile                     # Build multi-stage
├── docker-compose.yml             # Orquestación local
├── nginx.conf                     # Configuración Nginx
├── .env.example                   # Template variables
├── .dockerignore                  # Exclusiones Docker
├── package.json                   # Dependencias
├── vite.config.js                 # Config Vite + proxy API
├── tailwind.config.js             # Config Tailwind CSS
├── postcss.config.js              # Config PostCSS
└── README.md                       # Este archivo
```

---

## 🚀 Pipeline CI/CD

### Flujo Completo: Local → Docker Hub → ECR → EC2

```
┌─────────────────────────────────────────────────────────────┐
│  TRIGGER: Push en rama 'deploy'                            │
└────────────────────┬────────────────────────────────────────┘
                     ▼
        ┌────────────────────────┐
        │  1. Build Docker Image │  (ubuntu-latest)
        │  - Dockerfile compile  │
        │  - npm run build       │
        └────────┬───────────────┘
                 ▼
    ┌─────────────────────────────┐
    │  2. Push to Docker Hub      │
    │  - docker.io/user/ev2-...  │
    │  - Tags: latest, SHA        │
    └────────┬────────────────────┘
             ▼
    ┌──────────────────────────┐
    │  3. Push to AWS ECR      │
    │  - Pull desde Docker Hub │
    │  - Tag y push a ECR      │
    └────────┬─────────────────┘
             ▼
    ┌──────────────────────────┐
    │  4. Deploy en EC2        │
    │  - SSH a instancia Front │
    │  - Pull imagen ECR       │
    │  - docker run container  │
    │  - Clean old images      │
    └──────────────────────────┘
```

### Archivos necesarios

**Ubicación:** `.github/workflows/deploy.yml`

Pasos principales:

1. **build-and-push:**
   - Checkout código
   - Build imagen Docker
   - Push a Docker Hub con tags `latest` y `<commit-sha>`

2. **push-to-ecr:**
   - Configure credenciales AWS
   - Login a ECR
   - Pull imagen de Docker Hub
   - Tag como ECR image
   - Push a ECR (latest + commit-sha)

3. **deploy-to-ec2:**
   - SSH a instancia Frontend (IP privada o pública)
   - Login a ECR con credenciales AWS
   - Stop/remove contenedor anterior
   - Pull imagen nueva
   - Run contenedor con variables de entorno
   - Limpiar imágenes antiguas

4. **notify:**
   - Validar que deploy fue exitoso

---

## 🔐 GitHub Secrets Requeridos

**Ubicación:** Settings → Secrets and variables → Actions

| Secret | Valor | Ejemplo |
|--------|-------|---------|
| `DOCKER_HUB_USERNAME` | Tu usuario Docker Hub | `fabian02` |
| `DOCKER_HUB_TOKEN` | Token de acceso Docker Hub | `dckr_pat_xxx...` |
| `AWS_ACCESS_KEY_ID` | AWS Access Key | `AKIXXXXX...` |
| `AWS_SECRET_ACCESS_KEY` | AWS Secret Key | `wJalrXUtnFEMI/K7xxx...` |
| `EC2_FRONTEND_HOST` | IP/DNS de instancia Frontend | `ec2-1.amazonaws.com` |
| `EC2_USER` | Usuario SSH (ubuntu/ec2-user) | `ubuntu` |
| `EC2_PRIVATE_KEY` | Private key SSH (.pem) | `-----BEGIN RSA PRIVATE KEY-----...` |
| `BACKEND_API_URL` | URL pública del Backend | `http://10.0.1.50:8080` |

**Cómo crear secrets:**
1. Ve a tu repo → **Settings**
2. Sidebar: **Secrets and variables** → **Actions**
3. Click **New repository secret**
4. Nombre: `DOCKER_HUB_USERNAME`
5. Valor: tu usuario Docker Hub
6. Repite para cada secret

---

## 🖥️ Despliegue en EC2

### Prerequisitos en la Instancia

```bash
# Actualizar sistema
sudo apt update && sudo apt upgrade -y

# Instalar Docker
sudo apt install -y docker.io docker-compose

# Agregar usuario al grupo docker
sudo usermod -aG docker $USER
newgrp docker

# Instalar AWS CLI (para ECR)
sudo apt install -y awscli

# Configurar credenciales AWS
aws configure
```

### Deployment Manual (para testing)

```bash
# Login a ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  <account-id>.dkr.ecr.us-east-1.amazonaws.com

# Pull imagen
docker pull <account-id>.dkr.ecr.us-east-1.amazonaws.com/ev2-frontend:latest

# Run
docker run -d \
  --name ev2-frontend \
  --restart unless-stopped \
  -p 3000:80 \
  -e VITE_API_URL=http://backend-internal:8080 \
  <account-id>.dkr.ecr.us-east-1.amazonaws.com/ev2-frontend:latest

# Ver logs
docker logs ev2-frontend
```

### Acceso desde navegador

```
http://<IP-Pública-EC2>:3000
```

**Validar:**
- ✅ Página carga sin errores
- ✅ Console del navegador sin errores
- ✅ Puede comunicarse con Backend

---

## 🔍 Troubleshooting

### Docker: "Cannot connect to Docker daemon"

```bash
# Solución
sudo systemctl start docker
sudo usermod -aG docker $USER
```

### docker-compose: Port already in use

```bash
# Ver qué usa el puerto
lsof -i :3000

# Cambiar puerto en .env
FRONTEND_PORT=3001
```

### Contenedor se queda en loop de reinicio

```bash
# Ver logs
docker logs ev2-frontend

# Inspeccionar contenedor
docker inspect ev2-frontend
```

### Frontend no se comunica con Backend

1. Verificar `VITE_API_URL` en contenedor:
   ```bash
   docker inspect ev2-frontend | grep VITE_API_URL
   ```

2. Testear conectividad:
   ```bash
   docker exec ev2-frontend curl -v http://backend:8080/health
   ```

3. Revisar proxy en `vite.config.js` (solo para desarrollo)

### Pipeline CI/CD falla

1. Revisar logs en GitHub Actions:
   - Repo → **Actions** → último workflow
   - Click en job fallido
   - Ver detalles de cada paso

2. Problemas comunes:
   - ❌ GitHub Secrets incorrectos/incompletos
   - ❌ EC2 no accesible via SSH (revisar Security Groups)
   - ❌ ECR repository no existe (crear manualmente o vía IaC)
   - ❌ Docker Hub credenciales expiradas

---

## 📊 Decisiones Técnicas & Justificación

### Dockerfile Multi-Stage

**Decisión:** Usar 2 stages (Builder + Production)

**Justificación:**
- ✅ Imagen final reducida (~30MB vs ~500MB)
- ✅ No incluye node_modules en producción
- ✅ Mejora seguridad (menos código)
- ✅ Deploy más rápido

### Nginx en lugar de Node

**Decisión:** Servir con Nginx en Stage 2

**Justificación:**
- ✅ Ultra ligero (21MB imagen base)
- ✅ Rendimiento superior (C vs Node)
- ✅ SPA routing nativo con `try_files`
- ✅ Gzip compression built-in
- ✅ Caching de assets estáticos

### Docker Hub → ECR

**Decisión:** Publicar primero en Docker Hub, luego push a ECR

**Justificación:**
- ✅ Cache reutilizable entre builds
- ✅ Redundancia (backup en Docker Hub)
- ✅ Mejor latencia si hay múltiples regiones AWS
- ✅ Separación de concerns (CI en Docker Hub, CD en ECR)

### Named volumes vs Bind mounts

**Decisión:** No persistencia en Frontend (stateless)

**Justificación:**
- ✅ Frontend es stateless
- ✅ Cada deploy es nueva imagen
- ✅ Facilita scaling horizontal
- ✅ Simplifica CI/CD

---

## 📝 Commits Recomendados

```bash
# Estructura recomendada
git add .
git commit -m "feat: add Dockerfile with multi-stage build"
git commit -m "feat: add docker-compose for local development"
git commit -m "feat: add nginx configuration for SPA routing"
git commit -m "feat: add GitHub Actions CI/CD pipeline"
git commit -m "docs: add comprehensive README"
git commit -m "chore: add .dockerignore and .env.example"

# Push a rama deploy para triggear pipeline
git push origin deploy
```

---

## 🤝 Equipo

- **Desarrolladores:** Fabian Reyes
- **Empresa:** Innovatech Chile
- **Evaluación:** EP2 - Contenedorización y CI/CD

---

## 📞 Soporte

Para preguntas o issues:
1. Revisar GitHub Issues
2. Consultar la sección Troubleshooting
3. Revisar logs del pipeline en GitHub Actions
