# Config Repository - Sistema de Reservas de Hoteles

## ¿Qué es config-repo?

**config-repo NO es un servicio ni tiene puerto.** Es simplemente un **directorio que contiene archivos YAML** con las configuraciones de todos los microservicios.

El **Config Server** (puerto 8888) es quien lee estos archivos y los expone vía HTTP para que los microservicios los consuman.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   config-repo/                        Es solo un DIRECTORIO                 │
│   ├── api-gateway.yml                 con archivos de texto YAML            │
│   ├── auth-service.yml                                                      │
│   ├── hotel-service.yml               NO SE EJECUTA                         │
│   ├── reserva-service.yml             NO TIENE PUERTO                       │
│   └── notificacion-service.yml        NO ES UN SERVICIO                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              │  El Config Server LEE estos archivos
                              │  desde el filesystem (modo native)
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   CONFIG SERVER (Puerto 8888)         Este SÍ es un servicio                │
│                                                                             │
│   Lee los archivos YAML del config-repo y los expone via HTTP:              │
│                                                                             │
│   GET http://localhost:8888/api-gateway/default                             │
│   GET http://localhost:8888/auth-service/default                            │
│   GET http://localhost:8888/hotel-service/default                           │
│   ...                                                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              │  Los microservicios solicitan
                              │  su configuración al Config Server
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   MICROSERVICIOS                                                            │
│                                                                             │
│   Al iniciar, cada microservicio hace:                                      │
│   GET http://config-server:8888/{nombre-servicio}/default                   │
│                                                                             │
│   Y recibe su archivo YAML con toda la configuración                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Estructura del Directorio

```
C:\DAW_II\config-repo\
├── api-gateway.yml           # Configuración del API Gateway
├── auth-service.yml          # Configuración del Auth Service
├── hotel-service.yml         # Configuración del Hotel Service
├── reserva-service.yml       # Configuración del Reserva Service
├── notificacion-service.yml  # Configuración del Notificacion Service
└── README.md                 # Este archivo
```

## ¿Cómo se Conecta con Config Server?

El Config Server tiene en su `application.yml` la ruta a este directorio:

```yaml
# Config Server - application.yml
spring:
  profiles:
    active: native                    # Modo "native" = lee archivos locales
  cloud:
    config:
      server:
        native:
          search-locations: file:../config-repo   # ← Ruta a este directorio
```

**Modos de conexión:**

| Modo | Descripción | Configuración |
|------|-------------|---------------|
| **native** | Lee archivos del filesystem local | `file:../config-repo` o `file:/ruta/absoluta` |
| **git** | Clona un repositorio Git | `https://github.com/usuario/config-repo.git` |

### Modo Native (Desarrollo Local)

```yaml
spring:
  cloud:
    config:
      server:
        native:
          search-locations: file:../config-repo
```

### Modo Git (Producción)

```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/tu-usuario/config-repo.git
          default-label: main
          clone-on-start: true
```

## Archivos de Configuración

### api-gateway.yml

```yaml
server:
  port: ${SERVER_PORT:8080}

spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "http://localhost:4200"
            allowedMethods:
              - GET
              - POST
              - PUT
              - DELETE
              - OPTIONS
      routes:
        - id: auth-service
          uri: ${AUTH_SERVICE_URL:http://localhost:8081}
          predicates:
            - Path=/api/v1/auth/**
        # ... más rutas

eureka:
  client:
    service-url:
      defaultZone: ${EUREKA_URL:http://localhost:8761/eureka}
```

### auth-service.yml

```yaml
server:
  port: ${SERVER_PORT:8081}
  servlet:
    context-path: /api/v1

spring:
  datasource:
    url: ${SPRING_DATASOURCE_URL}
    username: ${SPRING_DATASOURCE_USERNAME}
    password: ${SPRING_DATASOURCE_PASSWORD}

  rabbitmq:
    host: ${SPRING_RABBITMQ_HOST}
    port: ${SPRING_RABBITMQ_PORT}

application:
  security:
    jwt:
      secret-key: ${JWT_SECRET_KEY}
      expiration: ${JWT_EXPIRATION}

eureka:
  client:
    service-url:
      defaultZone: ${EUREKA_URL:http://localhost:8761/eureka}
```

### hotel-service.yml

```yaml
server:
  port: ${SERVER_PORT:8082}
  servlet:
    context-path: /api/v1

spring:
  datasource:
    url: ${SPRING_DATASOURCE_URL}
    username: ${SPRING_DATASOURCE_USERNAME}
    password: ${SPRING_DATASOURCE_PASSWORD}

internal:
  auth-service:
    url: ${AUTH_SERVICE_URL:http://auth-service}

eureka:
  client:
    service-url:
      defaultZone: ${EUREKA_URL:http://localhost:8761/eureka}
```

### reserva-service.yml

```yaml
server:
  port: ${SERVER_PORT:8083}
  servlet:
    context-path: /api/v1

spring:
  datasource:
    url: ${SPRING_DATASOURCE_URL}
    username: ${SPRING_DATASOURCE_USERNAME}
    password: ${SPRING_DATASOURCE_PASSWORD}

  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

app:
  kafka:
    topics:
      reserva-notifications: ${KAFKA_RESERVA_NOTIFICATIONS_TOPIC:reserva.notifications}

internal:
  auth-service:
    url: ${AUTH_SERVICE_URL:http://auth-service}
  hotel-service:
    url: ${HOTEL_SERVICE_URL:http://hotel-service}
```

### notificacion-service.yml

```yaml
server:
  port: ${SERVER_PORT:8084}
  servlet:
    context-path: /api/v1

spring:
  datasource:
    url: ${SPRING_DATASOURCE_URL}
    username: ${SPRING_DATASOURCE_USERNAME}
    password: ${SPRING_DATASOURCE_PASSWORD}

  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    consumer:
      group-id: ${KAFKA_NOTIFICATION_GROUP_ID:notificacion-service}

  mail:
    host: smtp.gmail.com
    port: 587
    username: ${MAIL_USERNAME}
    password: ${MAIL_PASSWORD}
```

## Variables de Entorno

Cada archivo YAML usa variables de entorno con valores por defecto:

| Variable | Descripción | Default |
|----------|-------------|---------|
| `SERVER_PORT` | Puerto del servicio | Varía por servicio |
| `SPRING_DATASOURCE_URL` | URL de conexión MySQL | - |
| `SPRING_DATASOURCE_USERNAME` | Usuario de BD | - |
| `SPRING_DATASOURCE_PASSWORD` | Contraseña de BD | - |
| `SPRING_RABBITMQ_HOST` | Host de RabbitMQ | localhost |
| `SPRING_RABBITMQ_PORT` | Puerto de RabbitMQ | 5672 |
| `KAFKA_BOOTSTRAP_SERVERS` | Servidores Kafka | localhost:9092 |
| `EUREKA_URL` | URL de Eureka | http://localhost:8761/eureka |
| `JWT_SECRET_KEY` | Clave secreta JWT | - |
| `MAIL_USERNAME` | Email SMTP | - |
| `MAIL_PASSWORD` | Password SMTP | - |

---

## Flujo Completo de Configuración

```
PASO 1: Config Server inicia
════════════════════════════════════════════════════════════════════════════

   Config Server lee su application.yml:

   spring.cloud.config.server.native.search-locations: file:../config-repo
                                                              │
                                                              ▼
   Config Server escanea el directorio config-repo/
   y carga todos los archivos .yml en memoria


PASO 2: Microservicio inicia y solicita configuración
════════════════════════════════════════════════════════════════════════════

   Auth Service tiene en su application.yml:

   spring:
     application:
       name: auth-service              ◄── Este nombre es importante
     config:
       import: optional:configserver:http://localhost:8888
                                              │
                                              ▼
   Auth Service hace: GET http://localhost:8888/auth-service/default
                                              │
                                              ▼
   Config Server busca archivo: auth-service.yml
   y retorna su contenido como JSON


PASO 3: Microservicio aplica la configuración
════════════════════════════════════════════════════════════════════════════

   Auth Service recibe:
   {
     "server.port": "8081",
     "spring.datasource.url": "jdbc:mysql://...",
     "application.security.jwt.secret-key": "...",
     ...
   }

   Y aplica estas propiedades a su contexto Spring
```

---

## Docker

### ¿Cómo se monta config-repo en Docker?

Como config-repo es solo un directorio, se monta como **volumen** en el contenedor del Config Server:

```yaml
# docker-compose.yml
services:
  config-server:
    build: ./config-server
    ports:
      - "8888:8888"
    volumes:
      - ./config-repo:/config-repo:ro    # ← Monta el directorio como volumen
    environment:
      - SPRING_CLOUD_CONFIG_SERVER_NATIVE_SEARCH_LOCATIONS=file:/config-repo
```

### docker-compose.yml Completo

```yaml
version: '3.8'

services:
  # ============================================
  # CONFIG SERVER (lee config-repo)
  # ============================================
  config-server:
    build: ../config-server
    container_name: config-server
    ports:
      - "8888:8888"
    volumes:
      - ./:/config-repo:ro              # Monta este directorio
    environment:
      - SPRING_PROFILES_ACTIVE=native
      - SPRING_CLOUD_CONFIG_SERVER_NATIVE_SEARCH_LOCATIONS=file:/config-repo
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:8888/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - hotel-network

  # ============================================
  # DISCOVERY SERVICE
  # ============================================
  discovery-service:
    build: ../discovery-service
    container_name: discovery-service
    ports:
      - "8761:8761"
    depends_on:
      config-server:
        condition: service_healthy
    networks:
      - hotel-network

  # ============================================
  # API GATEWAY
  # ============================================
  api-gateway:
    build: ../api-gateway
    container_name: api-gateway
    ports:
      - "8080:8080"
    environment:
      - CONFIG_SERVER_URL=http://config-server:8888
      - EUREKA_URL=http://discovery-service:8761/eureka
    depends_on:
      - discovery-service
    networks:
      - hotel-network

  # ============================================
  # MICROSERVICIOS
  # ============================================
  auth-service:
    build: ../ms-auth/auth-service
    container_name: auth-service
    ports:
      - "8081:8081"
    environment:
      - CONFIG_SERVER_URL=http://config-server:8888
      - SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/auth_db
      - SPRING_DATASOURCE_USERNAME=hotel_user
      - SPRING_DATASOURCE_PASSWORD=hotel_pass
      - JWT_SECRET_KEY=tu-clave-secreta-256-bits
      - EUREKA_URL=http://discovery-service:8761/eureka
    depends_on:
      - config-server
      - mysql
    networks:
      - hotel-network

  hotel-service:
    build: ../ms-hotel/hotel-service
    container_name: hotel-service
    ports:
      - "8082:8082"
    environment:
      - CONFIG_SERVER_URL=http://config-server:8888
      - SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/hotel_db
      - SPRING_DATASOURCE_USERNAME=hotel_user
      - SPRING_DATASOURCE_PASSWORD=hotel_pass
      - EUREKA_URL=http://discovery-service:8761/eureka
    depends_on:
      - auth-service
    networks:
      - hotel-network

  reserva-service:
    build: ../ms-reserva/reserva-service
    container_name: reserva-service
    ports:
      - "8083:8083"
    environment:
      - CONFIG_SERVER_URL=http://config-server:8888
      - SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/reserva_db
      - SPRING_DATASOURCE_USERNAME=hotel_user
      - SPRING_DATASOURCE_PASSWORD=hotel_pass
      - KAFKA_BOOTSTRAP_SERVERS=kafka:29092
      - EUREKA_URL=http://discovery-service:8761/eureka
    depends_on:
      - hotel-service
      - kafka
    networks:
      - hotel-network

  notificacion-service:
    build: ../ms-notificacion/notificacion-service
    container_name: notificacion-service
    ports:
      - "8084:8084"
    environment:
      - CONFIG_SERVER_URL=http://config-server:8888
      - SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/notificacion_db
      - SPRING_DATASOURCE_USERNAME=hotel_user
      - SPRING_DATASOURCE_PASSWORD=hotel_pass
      - KAFKA_BOOTSTRAP_SERVERS=kafka:29092
      - MAIL_USERNAME=tu-email@gmail.com
      - MAIL_PASSWORD=tu-app-password
      - EUREKA_URL=http://discovery-service:8761/eureka
    depends_on:
      - reserva-service
    networks:
      - hotel-network

  # ============================================
  # INFRAESTRUCTURA
  # ============================================
  mysql:
    image: mysql:8.0
    container_name: mysql-hotel
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_USER: hotel_user
      MYSQL_PASSWORD: hotel_pass
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
      - ./init-db.sql:/docker-entrypoint-initdb.d/init-db.sql
    networks:
      - hotel-network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

  rabbitmq:
    image: rabbitmq:3-management
    container_name: rabbitmq-hotel
    ports:
      - "5672:5672"
      - "15672:15672"
    networks:
      - hotel-network

  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    container_name: zookeeper-hotel
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    networks:
      - hotel-network

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    container_name: kafka-hotel
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    networks:
      - hotel-network

networks:
  hotel-network:
    driver: bridge

volumes:
  mysql_data:
```

### init-db.sql

```sql
-- Crear bases de datos
CREATE DATABASE IF NOT EXISTS auth_db;
CREATE DATABASE IF NOT EXISTS hotel_db;
CREATE DATABASE IF NOT EXISTS reserva_db;
CREATE DATABASE IF NOT EXISTS notificacion_db;

-- Otorgar permisos
GRANT ALL PRIVILEGES ON auth_db.* TO 'hotel_user'@'%';
GRANT ALL PRIVILEGES ON hotel_db.* TO 'hotel_user'@'%';
GRANT ALL PRIVILEGES ON reserva_db.* TO 'hotel_user'@'%';
GRANT ALL PRIVILEGES ON notificacion_db.* TO 'hotel_user'@'%';

FLUSH PRIVILEGES;
```

---

## Kubernetes

En Kubernetes, el config-repo se convierte en un **ConfigMap**:

```yaml
# k8s/configmap-config-repo.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-repo
  namespace: hotel-system
data:
  api-gateway.yml: |
    server:
      port: ${SERVER_PORT:8080}
    spring:
      application:
        name: api-gateway
    # ... resto de la configuración

  auth-service.yml: |
    server:
      port: ${SERVER_PORT:8081}
    # ... resto de la configuración

  hotel-service.yml: |
    server:
      port: ${SERVER_PORT:8082}
    # ... resto de la configuración

  reserva-service.yml: |
    server:
      port: ${SERVER_PORT:8083}
    # ... resto de la configuración

  notificacion-service.yml: |
    server:
      port: ${SERVER_PORT:8084}
    # ... resto de la configuración
```

### Config Server Deployment con ConfigMap

```yaml
# k8s/config-server-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-server
  namespace: hotel-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config-server
  template:
    metadata:
      labels:
        app: config-server
    spec:
      containers:
        - name: config-server
          image: ${ACR_NAME}.azurecr.io/config-server:latest
          ports:
            - containerPort: 8888
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "native"
            - name: SPRING_CLOUD_CONFIG_SERVER_NATIVE_SEARCH_LOCATIONS
              value: "file:/config-repo"
          volumeMounts:
            - name: config-repo
              mountPath: /config-repo
              readOnly: true
      volumes:
        - name: config-repo
          configMap:
            name: config-repo           # ← Monta el ConfigMap como directorio
```

### Crear ConfigMap desde archivos locales

```bash
# Crear ConfigMap directamente desde los archivos YAML
kubectl create configmap config-repo \
  --namespace hotel-system \
  --from-file=api-gateway.yml=./api-gateway.yml \
  --from-file=auth-service.yml=./auth-service.yml \
  --from-file=hotel-service.yml=./hotel-service.yml \
  --from-file=reserva-service.yml=./reserva-service.yml \
  --from-file=notificacion-service.yml=./notificacion-service.yml
```

---

## Azure

### Opción 1: ConfigMap en AKS

Igual que en Kubernetes estándar, usar ConfigMap.

### Opción 2: Azure App Configuration

Azure tiene un servicio nativo para configuración:

```bash
# Crear Azure App Configuration
az appconfig create \
  --name appconfig-hotel-reservas \
  --resource-group rg-hotel-reservas \
  --location eastus \
  --sku Standard

# Importar archivos YAML
az appconfig kv import \
  --name appconfig-hotel-reservas \
  --source file \
  --path ./api-gateway.yml \
  --format yaml \
  --prefix "api-gateway/"

az appconfig kv import \
  --name appconfig-hotel-reservas \
  --source file \
  --path ./auth-service.yml \
  --format yaml \
  --prefix "auth-service/"

# ... repetir para cada servicio
```

### Opción 3: Repositorio Git

En producción, es común tener config-repo como repositorio Git:

```bash
# Crear repositorio en GitHub/Azure DevOps
git init
git add .
git commit -m "Initial config"
git remote add origin https://github.com/tu-usuario/config-repo.git
git push -u origin main
```

Config Server apunta al repositorio:

```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/tu-usuario/config-repo.git
          default-label: main
          clone-on-start: true
          # Para repos privados:
          username: ${GIT_USERNAME}
          password: ${GIT_PASSWORD}
```

---

## Resumen

| Pregunta | Respuesta |
|----------|-----------|
| ¿config-repo es un servicio? | **NO**, es solo un directorio con archivos |
| ¿Tiene puerto? | **NO**, no se ejecuta |
| ¿Cómo se conecta con Config Server? | Config Server **lee** los archivos del filesystem |
| ¿Dónde se configura la ruta? | En `application.yml` del Config Server |
| ¿En Docker? | Se monta como **volumen** |
| ¿En Kubernetes? | Se convierte en **ConfigMap** |
| ¿En producción? | Se usa un **repositorio Git** |

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   config-repo (directorio)  ──────►  Config Server (servicio)   │
│   Solo archivos YAML               Lee y expone via HTTP        │
│   NO tiene puerto                  Puerto 8888                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
