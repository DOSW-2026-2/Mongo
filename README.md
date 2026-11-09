# LABORATORIO 8 · APP ToDo CON MONGODB

**Curso:** Desarrollo y Operaciones de Software (DOSW)  
**Proyecto base:** App ToDo — Spring Boot + React + JWT + CI/CD  
**Modalidad:** Práctica corta por equipos, sobre el repositorio utilizado en el laboratorio de CI/CD  
**Entregable:** Repositorio GitHub actualizado, evidencias, informe breve y video

---

## 1. Reto

La aplicación **App ToDo** ya funciona con una API REST, React, autenticación JWT, PostgreSQL y pipelines de integración y despliegue. En esta práctica deberán **migrar únicamente la persistencia de las tareas a MongoDB**, una base de datos NoSQL orientada a documentos, y ejecutar **la misma aplicación** contra dos entornos:

1. **MongoDB local (on-premise simulado)** mediante la imagen oficial de Docker.
2. **MongoDB Atlas** mediante un clúster administrado en la nube.

No se desarrollará otro CRUD. Deben **conservar el CRUD y el contrato REST existente**, adaptar la capa de persistencia y demostrar que funciona en ambos entornos. **PostgreSQL podrá continuar almacenando usuarios y roles** del laboratorio de JWT; la colección de tareas deberá residir en MongoDB.

> **Alcance:** son **dos bases distintas**. Cambiar la conexión de local a Atlas **no sincroniza ni migra automáticamente** los datos. Se deberán crear datos de prueba en cada entorno.

### Arquitectura esperada

```mermaid
flowchart LR
    U[Usuario] --> R[React]
    R -->|HTTP + JWT| API[Spring Boot REST API]
    API --> S[TaskService]
    S --> MR[TaskMongoRepository]
    MR --> M{MONGO_URI}
    M --> L[(MongoDB Docker local)]
    M --> A[(MongoDB Atlas)]
    API --> AUTH[Seguridad / usuarios]
    AUTH --> PG[(PostgreSQL existente)]
```

**Solo se utilizará una conexión MongoDB por ejecución**, seleccionada mediante variables de entorno.

## 2. Requerimientos e historias de usuario

| ID | Requerimiento verificable |
|---|---|
| RF-01 | Crear, listar, consultar, editar y eliminar tareas en una colección `tasks` de MongoDB. |
| RF-02 | Mantener título, descripción, estado, prioridad, fecha límite y fecha de creación. |
| RF-03 | Mantener la autenticación JWT y permitir acceder únicamente a las tareas del usuario autenticado. |
| RF-04 | Ejecutar el backend con MongoDB local en Docker, conservando datos al reiniciar el contenedor. |
| RF-05 | Ejecutar el mismo backend con MongoDB Atlas cambiando la configuración, no la lógica del negocio. |
| RF-06 | Mantener pruebas automatizadas y actualizar el pipeline de integración del laboratorio anterior. |

**HU-01.** Como usuario autenticado, quiero gestionar mis tareas para conservarlas en MongoDB sin perder las funcionalidades existentes.  
**HU-02.** Como desarrollador, quiero alternar entre una base MongoDB local y Atlas para verificar la portabilidad de la aplicación.  
**HU-03.** Como usuario, quiero que otro usuario no pueda consultar ni modificar mis tareas, aunque tenga un token JWT válido.

**Criterios de aceptación:** el CRUD funciona desde React, los cambios se observan en MongoDB, el usuario B no accede a tareas del usuario A, y la aplicación funciona tanto con Docker como con Atlas.

---

## PARTE A · MongoDB local con Docker

### 3. Descargar y ejecutar MongoDB

Desde el equipo local, descargar la imagen oficial:

```bash
docker pull mongo:8.0
```

Crear el archivo `infra/docker-compose.mongo.yml`:

```yaml
services:
  mongo:
    image: mongo:8.0
    container_name: todo-mongo
    ports:
      - "127.0.0.1:27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_ROOT_USER:?Definir MONGO_ROOT_USER}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD:?Definir MONGO_ROOT_PASSWORD}
    volumes:
      - todo_mongo_data:/data/db

volumes:
  todo_mongo_data:
```

Crear `infra/.env` **solo en el computador local** (no subirlo a GitHub):

```dotenv
MONGO_ROOT_USER=todo_admin
MONGO_ROOT_PASSWORD=CAMBIAR_POR_UNA_CLAVE_LOCAL_SEGURA
```

Incluir `infra/.env` en `.gitignore`. Luego ejecutar desde la raíz del repositorio:

```bash
docker compose --env-file infra/.env -f infra/docker-compose.mongo.yml up -d
docker ps
```

Conectarse con la consola de MongoDB (solicitará la contraseña):

```bash
docker exec -it todo-mongo mongosh -u todo_admin --authenticationDatabase admin -p
```

Probar:

```javascript
use todo_nosql
db.runCommand({ ping: 1 })
show collections
```

> El puerto está asociado a `127.0.0.1`: la base local **no debe quedar expuesta a Internet**. El volumen conserva los documentos al detener o recrear el contenedor; no eliminen el volumen durante la demostración.

### 4. Adaptar la persistencia de Spring Boot

Agregar la dependencia **Spring Data MongoDB** al `backend/pom.xml` del proyecto existente:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

Crear un documento de tarea, ajustándolo a los nombres de las clases actuales:

```java
@Document(collection = "tasks")
public class TaskDocument {
    @Id
    private String id;
    private String ownerId;       // Identificador del usuario del JWT
    private String title;
    private String description;
    private TaskStatus status;
    private TaskPriority priority;
    private LocalDate dueDate;
    private LocalDateTime createdAt;
    // Constructores, getters y setters según su proyecto
}
```

Importar `@Document` de `org.springframework.data.mongodb.core.mapping.Document` y `@Id` de `org.springframework.data.annotation.Id`.

Crear o adaptar `TaskMongoRepository`:

```java
public interface TaskMongoRepository
        extends MongoRepository<TaskDocument, String> {

    List<TaskDocument> findByOwnerId(String ownerId);

    Optional<TaskDocument> findByIdAndOwnerId(String id, String ownerId);
}
```

**Actividades de implementación:**

1. Sustituir el acceso a `TaskRepository` basado en JPA por `TaskMongoRepository` **solo en la gestión de tareas**.
2. Actualizar `TaskService` y DTOs para que `id` sea `String`. **No cambiar las rutas** (`/api/v1/tasks/{id}`), aunque cambie el tipo interno del identificador.
3. Obtener `ownerId` desde el usuario autenticado, **nunca del JSON enviado por React**.
4. En consultar, editar y eliminar, filtrar por `id` **y** `ownerId`; retornar `404` cuando el usuario no sea propietario o la tarea no exista.
5. Mantener las validaciones, estados, prioridades, manejo de excepciones y pruebas de los laboratorios anteriores.
6. Mantener PostgreSQL/JPA únicamente para los módulos que todavía lo requieran, como usuarios y roles; evitar que existan **dos fuentes de verdad** para `tasks`.

> **Atención:** no es necesario migrar las tareas históricas de PostgreSQL para aprobar el laboratorio. Deben crear nuevas tareas de demostración en MongoDB. Si el frontend convierte los ID a número, corregirlo: MongoDB usa habitualmente identificadores tipo cadena.

### 5. Configurar y ejecutar contra MongoDB local

Configurar en el backend **la propiedad que corresponda a su versión de Spring Boot**:

**Spring Boot 3.x — `application.properties`:**

```properties
spring.data.mongodb.uri=${MONGO_URI}
```

**Spring Boot 4.x — `application.properties`:**

```properties
spring.mongodb.uri=${MONGO_URI}
```

No declarar ambas propiedades al mismo tiempo. Definir en el **entorno de ejecución local** una variable `MONGO_URI` con este formato (reemplazar usuario y contraseña):

```text
mongodb://todo_admin:SU_CLAVE_LOCAL@localhost:27017/todo_nosql?authSource=admin
```

Si Spring Boot se ejecuta **dentro de un contenedor conectado a la misma red Docker Compose**, sustituir `localhost` por el nombre del servicio `mongo`. La contraseña debe mantenerse fuera del repositorio y los caracteres especiales de una URI deben codificarse adecuadamente.

Ejecutar el backend y React como en el laboratorio anterior. Iniciar sesión, crear **al menos tres tareas**, editar una y eliminar otra. Verificar los documentos en `mongosh`:

```javascript
use todo_nosql
db.tasks.find({}, { title: 1, status: 1, ownerId: 1 }).pretty()
db.tasks.countDocuments({})
```

Apagar y volver a iniciar MongoDB, sin eliminar el volumen, y demostrar que las tareas permanecen.

---

## PARTE B · MongoDB Atlas

### 6. Crear una base de datos administrada

1. Entrar a [MongoDB Atlas](https://www.mongodb.com/cloud/atlas/register) y crear o ingresar a un proyecto.
2. Crear un clúster **Free (M0)**, si está disponible para el proyecto, en una región admitida. **No seleccionar un plan de pago** para este laboratorio.
3. Crear un **Database User** con permisos sobre la base `todo_nosql` (idealmente `readWrite` solo sobre esa base).
4. En **Network Access**, agregar la **IP pública desde la que se conectará el backend**. Para pruebas locales, autorizar la IP pública actual; para Azure/AWS, autorizar la IP pública de salida de la VM correspondiente.
5. En el clúster, elegir **Connect → Drivers → Java** y copiar la cadena de conexión `mongodb+srv://...`.
6. Completar usuario, contraseña y nombre de base de datos en la conexión, sin registrar estos valores en GitHub.

Ejemplo **ilustrativo, no funcional hasta completar los valores**:

```text
mongodb+srv://USUARIO:CLAVE@CLUSTER.mongodb.net/todo_nosql?retryWrites=true&w=majority
```

> Atlas utiliza TLS para la conexión; no desactivar la validación de certificados. Evitar abrir el acceso a `0.0.0.0/0`. Si la contraseña contiene caracteres especiales, aplicar codificación URL en la URI.

### 7. Ejecutar App ToDo contra Atlas

1. Detener el backend que está conectado a MongoDB local.
2. Reemplazar **únicamente** la variable `MONGO_URI` por la conexión de Atlas.
3. Volver a iniciar el backend y React. **No modificar Controllers, Services ni Repositories** para cambiar de proveedor.
4. Crear nuevamente tres tareas, editar una, eliminar otra y verificar los documentos desde **Atlas → Data Explorer → `todo_nosql` → `tasks`**.
5. Comprobar que las tareas creadas en la base local **no aparecen automáticamente** en Atlas.
6. Con dos usuarios distintos, verificar que un JWT válido **no permita consultar ni modificar las tareas del otro usuario**.

---

## PARTE C · Continuidad del laboratorio CI/CD

### 8. Actualizar integración y despliegue

**Pipeline `ci-quality.yml`:** conservar compilación, análisis estático, JaCoCo/Vitest y publicación de imágenes. Agregar un MongoDB de prueba al job de backend (sin eliminar PostgreSQL si aún es necesario para autenticación):

```yaml
services:
  mongo:
    image: mongo:8.0
    env:
      MONGO_INITDB_ROOT_USERNAME: ci_user
      MONGO_INITDB_ROOT_PASSWORD: ci_password_solo_pruebas
    ports:
      - 27017:27017
```

En el **step de Maven**, definir:

```yaml
env:
  MONGO_URI: mongodb://ci_user:ci_password_solo_pruebas@localhost:27017/todo_ci?authSource=admin
```

Los fragmentos se deben **integrar al job existente**: no reemplazar el bloque `services` completo si ya tiene el servicio PostgreSQL. Crear al menos una prueba de integración real del repositorio Mongo (guardar → consultar → eliminar), además de las pruebas unitarias con mocks. Verificar un PR con el pipeline aprobado.

**Pipelines `deploy-azure.yml` y `deploy-aws.yml`:** conservar las imágenes Docker y el proceso de despliegue existentes. Agregar `MONGO_URI` al entorno del contenedor backend en `infra/docker-compose.prod.yml`:

```yaml
services:
  backend:
    environment:
      MONGO_URI: ${MONGO_URI:?Falta MONGO_URI}
```

Este fragmento se **fusiona** con el servicio y sus variables existentes; no es un archivo completo. En cada VM guardar la conexión Atlas en el archivo `.env` protegido, como en el laboratorio anterior, y verificar que la IP de salida de esa VM esté autorizada en Atlas. **No incluir la URI real en el YAML, el README, capturas ni logs**. Mantener PostgreSQL en el despliegue si sigue siendo necesario para usuarios/JWT.

> Para este laboratorio corto **no es obligatorio volver a desplegar en las dos nubes** si no se dispone de créditos. Sí se debe dejar documentado el ajuste de los workflows y demostrar el funcionamiento local y en Atlas. Los despliegues anteriores deben seguir siendo reproducibles.

---

## 9. Entregables y evaluación

En el repositorio GitHub deben dejar:

- [ ] README actualizado con arquitectura, explicación de qué se guardó en MongoDB y cómo se selecciona el entorno.
- [ ] Archivos de configuración de Docker **sin contraseñas reales**.
- [ ] Modelo `TaskDocument`, `TaskMongoRepository`, servicio y pruebas actualizados.
- [ ] Pipeline CI ejecutado correctamente y CD adaptado para leer `MONGO_URI`.
- [ ] Capturas de **MongoDB Docker**, colección local, **MongoDB Atlas**, colección remota y pruebas.
- [ ] **Video de 4 a 6 minutos** mostrando el CRUD desde React contra MongoDB local y luego contra Atlas, incluyendo un caso de acceso denegado a tareas de otro usuario.
- [ ] `docs/informe-mongodb.md` con respuestas a las preguntas siguientes.

### Preguntas del informe

1. ¿Qué diferencias encontraron entre el modelo relacional de tareas y un documento MongoDB? Incluyan un ejemplo real de su colección.
2. ¿Qué cambiaron en `Entity`, `Repository` y `Service` al migrar las tareas de JPA a Spring Data MongoDB?
3. ¿Cómo garantizan que un usuario no pueda leer o modificar tareas ajenas en MongoDB?
4. ¿Qué diferencias encontraron entre MongoDB en Docker y MongoDB Atlas en conexión, administración, persistencia y exposición de red?
5. ¿Por qué bastó cambiar `MONGO_URI` para utilizar Atlas y qué cambios fueron necesarios en el pipeline CI/CD?

### Criterios de evaluación sugeridos

| Criterio | Peso |
|---|---:|
| Migración del CRUD de tareas y aislamiento por usuario | 30 % |
| MongoDB Docker: ejecución, persistencia y evidencia | 20 % |
| Atlas: conexión y CRUD desde React | 20 % |
| Pruebas y actualización del pipeline CI/CD | 20 % |
| README, informe y video | 10 % |
| **Total** | **100 %** |

### Referencias oficiales

- [MongoDB Docker (imagen oficial)](https://hub.docker.com/_/mongo)
- [MongoDB Atlas — crear clúster Free](https://www.mongodb.com/docs/atlas/tutorial/deploy-free-tier-cluster/)
- [MongoDB Atlas — conectividad y lista de IP](https://www.mongodb.com/docs/atlas/connect-to-database-deployment/)
- [Spring Boot — MongoDB](https://docs.spring.io/spring-boot/reference/data/nosql.html)
- [Spring Boot 4 — cambios de propiedades MongoDB](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide)

**Cierre:** el resultado debe demostrar que la misma App ToDo funciona con **dos despliegues distintos de MongoDB** sin modificar la lógica de negocio, sin perder la seguridad JWT y sin comprometer credenciales.
