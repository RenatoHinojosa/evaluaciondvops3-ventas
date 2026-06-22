# Backend Ventas (rama `deploy`)

README adaptado al estado real de la rama `deploy`.

## 1) Arquitectura backend (Java/Spring Boot)

Aplicación Spring Boot 3.4.x con Java 17 y arquitectura en capas:

- **Controller**: `src/main/java/com/citt/controller/VentaController.java`
  - Expone la API REST bajo `/api/v1/ventas`.
- **Service**:
  - Interfaz: `src/main/java/com/citt/persistence/services/VentaService.java`
  - Implementación: `src/main/java/com/citt/persistence/services/VentaServiceImpl.java`
- **Repository**: `src/main/java/com/citt/persistence/repository/VentaRepository.java` (Spring Data JPA).
- **Entidad JPA**: `src/main/java/com/citt/persistence/entity/Venta.java`.
- **Manejo de errores**:
  - Excepción de dominio: `VentaNotFoundException`
  - Handler global: `RestResponseEntityExceptionHandler` (retorna 404 con body estructurado).
- **OpenAPI/Swagger**:
  - Configuración: `src/main/java/com/citt/config/OpenApiConfing.java`
  - UI en `/swagger-ui.html`.

Dependencias clave (`pom.xml`):
- `spring-boot-starter-web`
- `spring-boot-starter-data-jpa`
- `mysql-connector-j` (runtime)
- `springdoc-openapi-starter-webmvc-ui`
- `spring-boot-starter-validation`
- `h2` (tests)

## 2) Configuración de MySQL vía variables de entorno

La app toma configuración desde `src/main/resources/application.properties`:

- `spring.datasource.url=jdbc:mysql://${DB_ENDPOINT}:${DB_PORT}/${DB_NAME}...`
- `spring.datasource.username=${DB_USERNAME}`
- `spring.datasource.******
- `spring.jpa.hibernate.ddl-auto=update`
- `server.port=8080`

Variables requeridas en runtime:

| Variable | Uso |
|---|---|
| `DB_ENDPOINT` | Host del servidor MySQL |
| `DB_PORT` | Puerto MySQL (normalmente `3306`) |
| `DB_NAME` | Nombre de base de datos |
| `DB_USERNAME` | Usuario |
| `DB_PASSWORD` | Contraseña |

### En Kubernetes (rama `deploy`)

En `k8s/deployment.yaml`, estas variables se inyectan así:

- `DB_ENDPOINT=ventas-mysql-service` (Service interno de MySQL)
- `DB_PORT=3306`
- `DB_NAME`, `DB_USERNAME`, `DB_PASSWORD` desde el Secret `ventas-db-secret`.

El Secret se crea/actualiza en CI desde GitHub Secrets.

## 3) Flujo exacto de CI/CD en `deploy` (GitHub Actions + ECR + EKS)

Workflow: `.github/workflows/main.yml`  
Trigger:
- `push` a rama `deploy`
- `workflow_dispatch` manual

### Job 1: `build-and-push`

1. Checkout.
2. Configura credenciales AWS (`aws-actions/configure-aws-credentials@v4`).
3. Login a ECR (`aws-actions/amazon-ecr-login@v2`).
4. Define `image_tag` = `${{ github.sha }}`.
5. Build Docker con 2 tags:
   - `${REGISTRY_URL}/${AWS_ECR_REPOSITORY}:${github.sha}`
   - `${REGISTRY_URL}/${AWS_ECR_REPOSITORY}:latest`
6. Push de ambos tags a ECR.

### Job 2: `deploy-to-eks` (depende del Job 1)

1. Checkout.
2. Configura credenciales AWS.
3. Instala `kubectl` (`v1.29.0`).
4. Ejecuta `aws eks update-kubeconfig` con `EKS_CLUSTER_NAME`.
5. Crea/aplica Secret `ventas-db-secret` en namespace objetivo (`EKS_NAMESPACE`) con:
   - `MYSQL_DATABASE` ← `DB_NAME`
   - `MYSQL_ROOT_PASSWORD` ← `DB_PASSWORD`
   - `MYSQL_USER` ← `DB_USER`
   - `MYSQL_PASSWORD` ← `DB_PASSWORD`
6. Aplica manifiestos `k8s/`:
   - `k8s/deployment.yaml`
   - `k8s/service.yaml`
   - `k8s/mysql-deployment.yaml`
   - `k8s/mysql-service.yaml`
7. Actualiza imagen del deployment (`kubectl set image`) usando:
   - `K8S_DEPLOYMENT_NAME`
   - `K8S_CONTAINER_NAME`
   - imagen ECR con `IMAGE_TAG` (SHA)
8. Verifica rollout y lista pods/services.

### Secrets usados por el workflow

- `AWS_ACCOUNT_ID`
- `AWS_REGION`
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_SESSION_TOKEN`
- `AWS_ECR_REPOSITORY`
- `EKS_CLUSTER_NAME`
- `EKS_NAMESPACE`
- `K8S_DEPLOYMENT_NAME`
- `K8S_CONTAINER_NAME`
- `DB_NAME`
- `DB_USER`
- `DB_PASSWORD`

## 4) Endpoints y documentación expuesta

Base path: `/api/v1/ventas`

| Método | Endpoint | Descripción |
|---|---|---|
| GET | `/api/v1/ventas` | Listar ventas |
| GET | `/api/v1/ventas/{idVenta}` | Obtener venta por ID |
| POST | `/api/v1/ventas` | Crear venta |
| PUT | `/api/v1/ventas/{idVenta}` | Actualizar venta |
| DELETE | `/api/v1/ventas/{idVenta}` | Eliminar venta |

Swagger/OpenAPI:
- UI: `http://localhost:8080/swagger-ui.html`

## 5) Guía práctica: clonar, configurar, ejecutar local y desplegar en EKS

## Clonar repositorio

```bash
git clone https://github.com/RenatoHinojosa/evaluaciondvops3-ventas.git
cd evaluaciondvops3-ventas
```

## Ejecutar local (Maven)

Definir variables:

```bash
export DB_ENDPOINT=localhost
export DB_PORT=3306
export DB_NAME=ventas
export DB_USERNAME=root
export DB_PASSWORD=tu_password
```

Luego:

```bash
mvn spring-boot:run
```

## Ejecutar en Docker local

```bash
docker build -t ventas-backend:local .
docker run --rm -p 8080:8080 \
  -e DB_ENDPOINT=host.docker.internal \
  -e DB_PORT=3306 \
  -e DB_NAME=ventas \
  -e DB_USERNAME=root \
  -e DB_PASSWORD=tu_password \
  ventas-backend:local
```

## Desplegar en EKS (flujo recomendado de la rama)

1. Configurar todos los GitHub Secrets del workflow.
2. Confirmar que el clúster EKS y el namespace existen.
3. Hacer push a la rama `deploy`.
4. Validar en Actions que:
   - la imagen se publicó en ECR,
   - el rollout del deployment `ventas` fue exitoso.

## Manifiestos Kubernetes (estado actual)

- `k8s/deployment.yaml`: Deployment de backend (`ventas`).
- `k8s/service.yaml`: Service `ClusterIP` (`ventas-service`).
- `k8s/mysql-deployment.yaml`: MySQL 8.0 en el clúster.
- `k8s/mysql-service.yaml`: Service interno `ventas-mysql-service`.

### Nota importante de persistencia

`k8s/mysql-deployment.yaml` usa `emptyDir`, por lo que **los datos de MySQL son efímeros** (se pierden si el pod se recrea).  
Para producción, reemplazar por un `PersistentVolumeClaim`.

## Dockerfile (build de la app)

`Dockerfile` multi-stage:

1. **Build stage**: `maven:3.9.6-eclipse-temurin-17`
   - `mvn clean package -DskipTests`
2. **Runtime stage**: `eclipse-temurin:17-jre-jammy`
   - Copia `target/*.jar` a `/app/app.jar`
   - Expone `8080`
   - `ENTRYPOINT ["java", "-jar", "app.jar"]`

## Observación de pruebas

El `contextLoads` puede fallar en entornos sin variables `DB_*` porque el perfil por defecto apunta a MySQL (`application.properties`).  
Existe configuración de test con H2 en `application-test.properties`, útil cuando se ejecutan pruebas con ese perfil.
