# Proyecto DevOps - Innovatech Chile

Sistema de microservicios containerizado con Docker para gestión de Despachos y Ventas de Innovatech Chile.

##  Descripción del Proyecto

Este proyecto implementa una solución DevOps completa con:
- **2 Microservicios Backend** (Spring Boot): API REST para Despachos y Ventas
- **1 Frontend** (React + Vite): Interfaz de usuario
- **Base de Datos**: MySQL 8.0
- **Orquestación**: Docker Compose
- **CI/CD**: GitHub Actions (automatización de build y deploy)

## Arquitectura

```
┌─────────────────────────────────────────────────────┐
│                    Internet (Public)                 │
└────────────────────────┬────────────────────────────┘
                         │
                    ┌────▼─────┐
                    │ Frontend  │ (Puerto 3000)
                    │ (Nginx)   │
                    └────┬─────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
   ┌────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐
   │ Backend    │ │ Backend     │ │   MySQL     │
   │ Despachos  │ │ Ventas      │ │   Database  │
   │ (8081)     │ │ (8082)      │ │ (3306)      │
   └────────────┘ └─────────────┘ └─────────────┘
   
   (Red Interna: innovatech-network)
```

## Inicio Rápido

### Requisitos Previos
- Docker Desktop instalado y ejecutándose
- Git
- Mínimo 4GB RAM disponible

### Instalación y Levantamiento

1. **Clonar el repositorio**
```bash
git clone https://github.com/tu-usuario/proyecto-devops.git
cd proyecto-devops
```

2. **Levantar todos los servicios**
```bash
docker-compose up -d
```

3. **Esperar a que los servicios inicien** (~30 segundos)
```bash
docker-compose ps
```

4. **Verificar que todo funciona**
- Frontend: http://localhost:3000
- Backend Despachos: http://localhost:8081
- Backend Ventas: http://localhost:8082
- MySQL: localhost:3306

### Detener los servicios
```bash
docker-compose down
```

## Estructura del Proyecto

```
proyecto-devops/
├── back-Despachos_SpringBoot/
│   └── Springboot-API-REST-DESPACHO/
│       ├── Dockerfile
│       ├── README.md
│       ├── pom.xml
│       ├── src/
│       └── application.properties
├── back-Ventas_SpringBoot/
│   └── Springboot-API-REST/
│       ├── Dockerfile
│       ├── README.md
│       ├── pom.xml
│       ├── src/
│       └── application.properties
├── front_despacho/
│   ├── Dockerfile
│   ├── nginx.conf
│   ├── README.md
│   ├── package.json
│   ├── vite.config.js
│   └── src/
├── docker-compose.yml
└── README.md (este archivo)
```

## Docker & Docker Compose

### Dockerfiles (Multi-Stage)

Cada backend usa **multi-stage build** para optimizar:
- **Stage 1 (builder)**: Compila con Maven (pesado)
- **Stage 2 (runtime)**: Ejecuta JAR con JRE (ligero)

**Ventajas:**
- Imágenes más pequeñas (~200MB vs 1GB)
- Mayor seguridad (sin herramientas de compilación en producción)
- Mejor rendimiento de despliegue

### Seguridad en Dockerfiles

-  Usuario no-root (`appuser`) - evita ataques de escalamiento de privilegios
-  Imágenes base slim (alpine) - menos superficie de ataque
-  Health checks - monitoreo automático de servicios

### Variables de Entorno

Por defecto (desarrollo local):
```
MYSQL_DATABASE: innovatech_db
MYSQL_USER: admin
MYSQL_PASSWORD: admin123
DB_ENDPOINT: mysql
DB_PORT: 3306
```

En **AWS RDS** (producción):
```
DB_ENDPOINT: mydb.xxxxx.rds.amazonaws.com
DB_PORT: 3306
(Especificar en GitHub Secrets o variables de entorno)
```

## Persistencia de Datos

### Volúmenes Docker

```yaml
volumes:
  mysql_data:
    driver: local
    mount_path: /var/lib/mysql
```

**¿Qué sucede si reinicio un contenedor?**
- Los datos persisten en el volumen `mysql_data`
- La base de datos NO se pierde

##  Comunicación entre Servicios

### Red Interna (innovatech-network)

Los servicios se comunican por nombre DNS interno:

```
Frontend → Backend-Despachos: http://backend-despachos:8080
Frontend → Backend-Ventas:    http://backend-ventas:8080
Backends → MySQL:             jdbc:mysql://mysql:3306/innovatech_db
```

### Puertos Mapeados

| Servicio | Puerto Interno | Puerto Externo |
|----------|----------------|----------------|
| Frontend | 80 (Nginx) | 3000 |
| Backend Despachos | 8080 | 8081 |
| Backend Ventas | 8080 | 8082 |
| MySQL | 3306 | 3306 |

## Comandos Útiles

```bash
# Ver estado de contenedores
docker-compose ps

# Ver logs de un servicio
docker-compose logs backend-despachos
docker-compose logs -f mysql  # -f = seguir en tiempo real

# Ejecutar comando dentro de un contenedor
docker-compose exec backend-despachos bash

# Reconstruir imágenes (si cambias código)
docker-compose build

# Reiniciar un servicio
docker-compose restart backend-ventas

# Limpiar todo (volúmenes, imágenes, contenedores)
docker-compose down -v
```

## Seguridad

### Mejores Prácticas Implementadas

1. **Contenedores sin root**: Todos usan usuario `appuser`
2. **Imágenes minimales**: Alpine/Slim para reducir vulnerabilidades
3. **Health checks**: Monitoreo automático de servicios
4. **Redes internas**: Base de datos NO expuesta a Internet
5. **Secrets en GitHub**: Credenciales AWS encriptadas

## CI/CD con GitHub Actions

### Pipeline (cuando esté configurado)

```
1. Push a rama 'deploy'
   ↓
2. GitHub Actions: Compilar + Crear imágenes Docker
   ↓
3. Publicar en Docker Hub
   ↓
4. Conectar a EC2 por SSH
   ↓
5. Descargar imágenes nuevas
   ↓
6. Ejecutar docker-compose up
   ↓
7. Sistema actualizado en producción
```

## Testing Local

### Verificar que Frontend se conecta a Backend

```bash
# 1. Ir a http://localhost:3000 en el navegador
# 2. Abrir DevTools (F12)
# 3. Ir a pestaña Network
# 4. Realizar una acción que haga request al backend
# 5. Verificar que la request llega a http://backend-despachos:8080 o http://backend-ventas:8080
```

### Verificar conectividad de Backend a MySQL

```bash
docker-compose exec backend-despachos bash
# Dentro del contenedor:
curl http://localhost:8080/health  # o tu endpoint de health check
```

## Documentación Adicional

- [Backend Despachos](./back-Despachos_SpringBoot/Springboot-API-REST-DESPACHO/README.md)
- [Backend Ventas](./back-Ventas_SpringBoot/Springboot-API-REST/README.md)
- [Frontend](./front_despacho/README.md)

## Equipo

- **Integrante 1**: Responsable de Dockerization y CI/CD
- **Integrante 2**: Responsable de Arquitectura y Presentación

## Cronograma

- **Semana X**: Entrega del Encargo (Dockerfiles, docker-compose, GitHub Actions)
- **Semana X+1**: Presentación técnica y defensa

## Próximos Pasos

1. Dockerizar Frontend y Backends
2. Configurar docker-compose
3. Implementar GitHub Actions workflow
4. Desplegar en AWS EC2
5. Presentación técnica

##  Preguntas Frecuentes

**P: ¿Por qué multi-stage en Dockerfile?**
A: Reduce el tamaño final de la imagen eliminando herramientas de compilación que no necesita en producción.

**P: ¿Qué pasa si MySQL falla?**
A: Los backends reintentan conectarse automáticamente gracias a `depends_on` y health checks.

**P: ¿Cómo agrego una nueva variable de entorno?**
A: Edita `docker-compose.yml` en la sección `environment:` del servicio y redeploy.

##  Soporte

Para dudas técnicas, contactar a los integrantes del equipo o revisar la [documentación oficial de Docker](https://docs.docker.com/).

---

**Última actualización**: Mayo 2026
**Versión**: 1.0