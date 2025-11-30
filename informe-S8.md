# Informe de Práctica: Despliegue Automatizado de Backend con Docker Compose

## 1\. Introducción y Objetivo

El objetivo de esta práctica fue automatizar el despliegue de una arquitectura de backend completa utilizando **Docker Compose**. Esto incluyó la contenerización de una aplicación Spring Boot (Java), la base de datos **PostgreSQL** y su interfaz de administración, **pgAdmin**.

La práctica se centró en la creación de un entorno local robusto que garantice la **persistencia de datos**, la **comunicación segura** entre servicios y la **optimización** de la imagen del backend mediante la técnica de *Multi-stage Build*.

-----

## 2\. Configuración de Arquitectura con Docker Compose

La infraestructura completa de tres servicios fue definida en el archivo `docker-compose.yml`, con las variables sensibles gestionadas en un archivo `.env` para garantizar la seguridad.

### 2.1. Archivos de Configuración

**`./.env` (Variables de Entorno)**

```text
# Variables para PostgreSQL
POSTGRES_USER=appuser
POSTGRES_PASSWORD=strongpassword
POSTGRES_DB=security_db

# Variables para pgAdmin
PGADMIN_DEFAULT_EMAIL=admin@example.com
PGADMIN_DEFAULT_PASSWORD=adminpass

# Configuración de Conexión para el Backend
DB_HOST=db
DB_PORT=5432
DB_NAME=${POSTGRES_DB}
DB_USER=${POSTGRES_USER}
DB_PASSWORD=${POSTGRES_PASSWORD}
```

**`./docker-compose.yml` (Orquestación)**

```yaml
version: '3.8'

services:
  # 1. Base de Datos (PostgreSQL)
  db:
    image: postgres:14-alpine
    container_name: postgres_db
    restart: always
    environment:
      POSTGRES_USER: ${POSTES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - backend-network

  # 2. Panel de Administración (pgAdmin)
  pgadmin:
    image: dpage/pgadmin4
    container_name: pgadmin_panel
    restart: always
    environment:
      PGADMIN_DEFAULT_EMAIL: ${PGADMIN_DEFAULT_EMAIL}
      PGADMIN_DEFAULT_PASSWORD: ${PGADMIN_DEFAULT_PASSWORD}
    ports:
      - "5050:80"
    networks:
      - backend-network
    depends_on:
      - db

  # 3. Aplicación Backend (Spring Boot)
  backend:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: spring_backend
    restart: always
    ports:
      - "8080:8080"
    networks:
      - backend-network
    depends_on:
      - db
    environment:
      # Conexión usando el nombre del servicio 'db'
      SPRING_DATASOURCE_URL: jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}
      SPRING_DATASOURCE_USERNAME: ${DB_USER}
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD}

volumes:
  postgres_data:
    driver: local

networks:
  backend-network:
    driver: bridge
```

### 2.2. Elementos Clave y Propósito

| Componente | Configuración Clave | Propósito |
| :--- | :--- | :--- |
| **`db`** | `postgres:14-alpine` | Provee el servicio de base de datos PostgreSQL, utilizando una imagen ligera. |
| **`backend-network`** | `driver: bridge` | Red personalizada que aísla los tres contenedores y les permite comunicarse usando los **nombres de servicio** (ej: `backend` se conecta a `db`). |
| **`postgres_data`** | `volumes` | **Persistencia de Datos**. Asegura que la base de datos PostgreSQL no se pierda al detener o eliminar el contenedor. |
| **`backend`** | `depends_on: db` | Asegura que el servicio `db` sea iniciado antes de intentar iniciar la aplicación backend, evitando errores de conexión. |
| **Variables** | `.env` | Gestión centralizada de credenciales y configuraciones de conexión para mantener el archivo `docker-compose.yml` limpio. |

-----

## 3\. Contenerización del Backend (Spring Boot)

Se implementó un `Dockerfile` en **Múltiples Etapas** para construir y optimizar la imagen final del backend, que es una aplicación Java/Spring Boot basada en Maven.

### 3.1. Dockerfile Multi-Stage

```dockerfile
# Etapa 1: CONSTRUCCIÓN (BUILDER STAGE)
FROM maven:3.8.6-openjdk-17 AS builder
WORKDIR /app
# ... [Pasos para descargar dependencias y compilar el JAR] ...
RUN mvn package -DskipTests

# Etapa 2: EJECUCIÓN (RUNTIME STAGE)
FROM openjdk:17-jre-alpine # Imagen base mínima de JRE (Java Runtime Environment)
WORKDIR /app

# Copia el JAR final desde la etapa 'builder' y descarta las herramientas de Maven
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 3.2. Optimización y Aprendizaje

| Técnica | Descripción | Ventaja en la Automatización |
| :--- | :--- | :--- |
| **Multi-Stage Build** | Utilizar dos imágenes `FROM`. La primera (*builder*) compila el código (Maven/JDK). La segunda (*runtime*) solo ejecuta el JAR compilado (JRE Alpine). | **Reducción de Tamaño:** La imagen final solo contiene el JRE (pequeño) y el JAR. El peso de Maven y el código fuente se descarta, optimizando el tiempo de descarga y despliegue.  |
| **Caché de Dependencias** | Copiar el `pom.xml` antes del código fuente y ejecutar `mvn dependency:go-offline`. | **Acelera la Reconstrucción:** Si solo cambia el código fuente y no las dependencias (pom.xml), Docker reutiliza la capa de instalación de dependencias, reduciendo el tiempo de *build*. |

-----

## 4\. Demostración y Verificación

La arquitectura fue levantada usando el comando `docker-compose up --build -d`.

### 4.1. Verificación de Servicios

1.  **pgAdmin:** Se accedió a `http://localhost:5050` y se configuró una conexión al host `db` (el nombre del servicio PostgreSQL). La conexión fue exitosa, confirmando la interconexión de la red.
2.  **Backend (Spring Boot):** La aplicación fue accesible en `http://localhost:8080`. Los logs del contenedor confirmaron la conexión exitosa a PostgreSQL utilizando el nombre de servicio `db` como `SPRING_DATASOURCE_URL`, lo cual verificó que el backend se inicializó correctamente con la base de datos.
3.  **Persistencia:** Tras eliminar los contenedores (`docker-compose down`) y volver a levantarlos (`docker-compose up -d`), los datos creados en PostgreSQL persistieron gracias al volumen `postgres_data`.

## 5\. Conclusión

Esta práctica demostró la capacidad de Docker Compose para orquestar entornos complejos de producción con un alto grado de automatización. El uso de **redes personalizadas** garantiza la comunicación segura y el **Multi-stage Build** optimiza el tamaño de la imagen del backend de Spring Boot. El resultado es un proceso de despliegue reproducible, eficiente y listo para ser utilizado en cualquier ambiente.
