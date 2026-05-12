# 🚀 Proyecto Semestral DevOps - ISY1101

**Innovatech Chile** | Etapa 2: Contenedorización y Despliegue Automatizado en AWS

## 👥 Integrantes
- Gerardo Vera
- Abihail Barría

---

## 📐 Arquitectura

```
Internet
    │
    ▼
[EC2 Frontend - Subred Pública]
  └─ Nginx:80 → React App
       │
       │ HTTP (Security Group: solo desde Frontend)
       ▼
[EC2 Backend - Subred Privada]
  ├─ ventas-api:8081  ──► db-ventas:3306   (named volume: ventas_db_data)
  └─ despachos-api:8082 ► db-despachos:3306 (named volume: despachos_db_data)
```

- **Solo el Frontend** es accesible desde Internet (puerto 80)
- El Backend vive en subred privada, protegido por Security Groups
- Las bases de datos NO tienen puertos expuestos al host

---

## 🐳 Contenedorización

### Multi-stage builds

Todos los servicios usan **multi-stage build** para reducir el tamaño de imagen y evitar incluir herramientas de desarrollo en producción:

| Servicio | Stage Build | Stage Producción | Tamaño aprox. |
|---|---|---|---|
| Frontend | `node:20-alpine` | `nginx:1.25-alpine` | ~25 MB |
| Ventas API | `maven:3.9-alpine` | `eclipse-temurin:17-jre-alpine` | ~85 MB |
| Despachos API | `maven:3.9-alpine` | `eclipse-temurin:17-jre-alpine` | ~85 MB |

### Principio de mínimo privilegio

Los contenedores de producción crean un usuario sin privilegios (`appuser` / `appgroup`) y lo usan para ejecutar el proceso. Si el proceso fuese comprometido, el atacante no tendría acceso root al contenedor ni al host EC2.

---

## 💾 Persistencia de datos

Se usan **named volumes** gestionados por Docker:

```yaml
volumes:
  ventas_db_data:    # Datos MySQL del microservicio Ventas
  despachos_db_data: # Datos MySQL del microservicio Despachos
```

### ¿Por qué named volume y no bind mount?

| Criterio | Named Volume ✅ | Bind Mount |
|---|---|---|
| Portabilidad | Docker lo gestiona, no depende del host | Requiere rutas absolutas del host |
| Rendimiento | Óptimo en Linux (kernel nativo) | Puede ser menor |
| Producción AWS | Recomendado | Solo para desarrollo local |
| Gestión | `docker volume` commands | Manual |

Los datos persisten aunque el contenedor se elimine. Solo se pierden si se ejecuta `docker volume rm`.

---

## ⚙️ Ejecución local

### Requisitos
- Docker Desktop (o Docker Engine en Linux)
- Git

### Pasos

```bash
# 1. Clonar el repositorio
git clone <URL_DEL_REPOSITORIO>
cd proyecto-semestral

# 2. Crear archivo de variables de entorno
cp .env.example .env
# Editar .env con tus valores

# 3. Levantar el stack completo
docker compose up -d

# 4. Verificar que todos los servicios están corriendo
docker compose ps

# 5. Acceder a los servicios
# Frontend:  http://localhost
# Ventas:    http://localhost:8081/actuator/health
# Despachos: http://localhost:8082/actuator/health
```

### Comandos útiles

```bash
# Ver logs de un servicio
docker compose logs -f ventas

# Reiniciar un servicio sin perder datos
docker compose restart ventas

# Detener sin eliminar volúmenes
docker compose stop

# Eliminar contenedores (mantiene volúmenes/datos)
docker compose down

# Eliminar TODO incluyendo datos
docker compose down -v
```

---

## 🔄 Pipeline CI/CD (GitHub Actions)

El pipeline se activa automáticamente con cada **push a la rama `deploy`**.

### Flujo completo

```
git push → rama deploy
    │
    ▼
Job 1: build-and-push
    ├── Checkout código
    ├── Setup Docker Buildx
    ├── Login Docker Hub
    ├── Build + Push → frontend-despacho:latest
    ├── Build + Push → ventas-api:latest
    └── Build + Push → despachos-api:latest
    │
    ▼ (solo si Job 1 fue exitoso)
Job 2: deploy
    ├── Copiar docker-compose.yml → EC2 (SCP)
    ├── SSH → EC2
    ├── docker pull (3 imágenes)
    ├── docker compose stop/rm (solo APIs, preserva BDs)
    ├── docker compose up -d
    └── docker image prune -f
```

### GitHub Secrets requeridos

> Configurar en: Repositorio → Settings → Secrets and variables → Actions

| Secret | Descripción |
|---|---|
| `DOCKER_USERNAME` | Usuario de Docker Hub |
| `DOCKER_PASSWORD` | Token de acceso de Docker Hub |
| `EC2_HOST` | IP pública de la instancia EC2 |
| `EC2_KEY` | Contenido completo del archivo `.pem` (clave privada SSH) |
| `MYSQL_ROOT_PASSWORD` | Password root de MySQL |
| `MYSQL_DB_VENTAS` | Nombre de la base de datos de ventas |
| `MYSQL_USER_VENTAS` | Usuario MySQL para ventas |
| `MYSQL_PASS_VENTAS` | Password MySQL para ventas |
| `MYSQL_DB_DESPACHOS` | Nombre de la base de datos de despachos |
| `MYSQL_USER_DESPACHOS` | Usuario MySQL para despachos |
| `MYSQL_PASS_DESPACHOS` | Password MySQL para despachos |

---

## 🛠️ Stack tecnológico

| Tecnología | Uso |
|---|---|
| **React + Vite** | Frontend SPA |
| **Nginx 1.25** | Servidor web para el build estático |
| **Spring Boot 3** | Microservicios backend |
| **Java 17** | Runtime de los backends |
| **MySQL 8** | Base de datos relacional |
| **Docker** | Contenedorización |
| **Docker Compose** | Orquestación local |
| **GitHub Actions** | CI/CD automatizado |
| **AWS EC2** | Infraestructura cloud |

---

## 🔒 Seguridad

- Contenedores ejecutan con **usuario no root**
- Bases de datos **no expuestas al host** (solo `expose`, no `ports`)
- Credenciales manejadas con **GitHub Secrets** (nunca en el código)
- Archivo `.env` excluido del repositorio vía `.gitignore`
- Security Groups de AWS restringen acceso entre subredes

---

## 📦 Estructura del repositorio

```
proyecto-semestral/
├── .github/
│   └── workflows/
│       └── deploy.yml          # Pipeline CI/CD
├── front_despacho/
│   ├── Dockerfile              # Multi-stage: node → nginx
│   ├── nginx.conf              # Configuración Nginx SPA
│   ├── src/                    # Código React
│   └── package.json
├── back-Ventas_SpringBoot/
│   └── Springboot-API-REST-VENTA/
│       ├── Dockerfile          # Multi-stage: maven → jre
│       └── src/
├── back-Despachos_SpringBoot/
│   └── Springboot-API-REST-DESPACHO/
│       ├── Dockerfile          # Multi-stage: maven → jre
│       └── src/
├── docker-compose.yml          # Stack completo
├── .env.example                # Template de variables
├── .gitignore
└── README.md
```
