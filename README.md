# Backend Ventas - Sistema de Gestión de Despachos y Ventas

##  Descripción del Proyecto

Este proyecto es el **Módulo de Ventas** del Sistema de Gestión de Despachos y Ventas. Es un microservicio encargado de la gestión de productos, pedidos y transacciones comerciales. 
Este backend funciona de manera desacoplada y se comunica con un Frontend en React y otro microservicio de Despachos, a través de un reverse proxy (Nginx).

El sistema está diseñado con propósitos **académicos y educativos** para demostrar buenas prácticas en el desarrollo de aplicaciones modernas con arquitectura de microservicios, bases de datos en la nube (AWS RDS) y despliegues mediante pipelines CI/CD.

---

##  Tecnologías Utilizadas

- **Spring Boot 3** - Framework principal para la creación del microservicio en Java.
- **Spring Data JPA & Hibernate** - ORM para la interacción con la base de datos.
- **MySQL** - Base de datos relacional (alojada en AWS RDS).
- **Swagger / OpenAPI 3** - Para la documentación interactiva de la API.
- **Docker** - Containerización de la aplicación.
- **GitHub Actions** - Pipeline CI/CD para automatizar builds y despliegues en AWS ECR y EC2.
- **Maven** - Herramienta de gestión de dependencias y build.

---

##  Configuración y Puerto

El servicio está configurado para ejecutarse localmente y en el contenedor en el puerto **8080**.
La conexión a la base de datos se realiza a través de variables de entorno para garantizar la seguridad de las credenciales (AWS RDS).

### Variables de Entorno Requeridas:
- `DB_ENDPOINT`: Endpoint de la base de datos MySQL en AWS RDS.
- `DB_PORT`: Puerto de la base de datos (por defecto 3306).
- `DB_NAME`: Nombre de la base de datos.
- `DB_USERNAME`: Usuario de la base de datos.
- `DB_PASSWORD`: Contraseña de la base de datos.

---

##  Endpoints Principales

La API RESTful está expuesta bajo el prefijo `/api/v1/ventas`.

| Método | Endpoint | Descripción |
| :--- | :--- | :--- |
| `GET` | `/api/v1/ventas` | Obtener todas las ventas registradas. |
| `GET` | `/api/v1/ventas/{idVenta}` | Obtener una venta específica por su ID. |
| `POST` | `/api/v1/ventas` | Crear un nuevo registro de venta. |
| `PUT` | `/api/v1/ventas/{idVenta}` | Actualizar la información de una venta existente. |
| `DELETE` | `/api/v1/ventas/{idVenta}` | Eliminar una venta por su ID. |

### Documentación de la API (Swagger)
Puedes probar e interactuar con la API directamente a través de Swagger UI cuando el servidor esté corriendo:
- **URL local:** `http://localhost:8080/swagger-ui.html`

---

##  Despliegue CI/CD

Al igual que el Frontend y el microservicio de Despachos, este servicio cuenta con un flujo CI/CD configurado con **GitHub Actions**.

### **Flujo del Pipeline:**
1. **Build & Push:** Construye el proyecto con Maven, empaqueta el `.jar` en una imagen Docker y la sube a **AWS ECR**.
2. **Deploy to EC2:** Se conecta a la instancia EC2 usando **AWS Systems Manager (SSM)**, descarga la nueva imagen desde ECR, detiene el contenedor antiguo y levanta el nuevo contenedor de ventas mapeando el puerto 8080.

---

##  Arquitectura del Sistema

Este microservicio forma parte de una arquitectura mayor enrutada por **Nginx**:
- **Frontend (React)**: Interfaz de usuario servida en el puerto 80.
- **Backend Ventas (Este proyecto)**: API en el puerto 8080 (`/api/v1/ventas/*`).
- **Backend Despachos**: API en el puerto 8081 (`/api/v1/despachos/*`).
- Todas las peticiones del cliente son manejadas por el Proxy Inverso (Nginx).
