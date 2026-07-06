# INFORME TÉCNICO: EVALUACIÓN FINAL TRANSVERSAL (EFT)

**Asignatura:** Introducción a Herramientas DevOps (ISY1101)  
**Semana de inicio:** 18  
**Carrera:** Ingeniería en Informática / Ingeniería en Conectividad y Redes  
**Institución:** Duoc UC  

---

## 1. Introducción y Resumen Ejecutivo

El presente informe técnico expone la solución de arquitectura, contenerización y automatización del ciclo de Integración y Entrega Continua (CI/CD) diseñada para la plataforma semestral de Ventas y Despachos. El objetivo principal de este encargo es migrar una aplicación compuesta por un Frontend (React) y dos Backends (APIs en Spring Boot) a un modelo de despliegue en la nube de AWS que sea escalable, seguro y automatizado, cumpliendo con los estándares y mejores prácticas de la cultura DevOps.

A lo largo del documento, se detallan las decisiones de diseño arquitectónico, el endurecimiento de contenedores Docker, la infraestructura de AWS (VPC, ECR, EC2, IAM) y la justificación de la orquestación serverless.

---

## 2. Método de Integración y Comunicación del Sistema

La plataforma se ha desacoplado en componentes autónomos que se comunican mediante protocolos estándar de la web:

1. **Frontend (Capa de Presentación):** 
   * Construido sobre React y Vite, se sirve al cliente a través de un servidor Nginx ligero.
   * Realiza peticiones asíncronas HTTP (REST) mediante la librería Axios al navegador del cliente hacia los puertos expuestos de los backends.
   * **Desacoplamiento de URLs:** Para evitar el antipatrón de IPs estáticas hardcodeadas, se centralizó la configuración en el módulo `config.js` inyectando variables de entorno de Vite (`import.meta.env.VITE_API_VENTAS_URL` y `import.meta.env.VITE_API_DESPACHOS_URL`).
2. **Backends (Capa de Lógica de Negocio):**
   * Dos APIs REST independientes basadas en Spring Boot 3.x expuestas en los puertos `8080` (Ventas) y `8081` (Despachos).
   * Cuentan con soporte de CORS (`@CrossOrigin(origins = "*")`) habilitado en sus controladores para permitir que el frontend cargado en el navegador realice peticiones cross-origin de manera controlada.
3. **Base de Datos Relacional (Capa de Datos):**
   * Motor MySQL relacional. Los backends se conectan utilizando la URL JDBC `jdbc:mysql://${DB_ENDPOINT}:${DB_PORT}/${DB_NAME}`.
   * Se configuró el driver JDBC con `createDatabaseIfNotExist=true` para permitir que el driver cree la base de datos de manera automática en el primer arranque. Hibernate se encarga de la auto-generación de tablas con `spring.jpa.hibernate.ddl-auto=update`.

---

## 3. Contenerización y Buenas Prácticas (Docker & Docker Compose)

Tanto el frontend como los backends se ejecutan dentro de contenedores optimizados para producción bajo las siguientes prácticas de endurecimiento y eficiencia:

### 3.1. Estructura de Dockerfiles Multietapa
El uso de builds multietapa (multi-stage builds) separa el entorno de construcción del entorno de ejecución final, garantizando imágenes ligeras y sin herramientas innecesarias:

* **Dockerfiles de Spring Boot (Ventas y Despachos):**
  * **Etapa 1 (Build):** Usa una imagen base pesada `maven:3.8.5-openjdk-17-slim` con todas las dependencias necesarias para compilar el código fuente. Se compila con `mvn package -DskipTests` para acelerar el pipeline y no requerir conexión a base de datos durante la construcción.
  * **Etapa 2 (Run):** Usa `eclipse-temurin:17-jre-alpine` (una JRE mínima basada en Alpine Linux).
  * **Endurecimiento de Seguridad:** Se crea un grupo y usuario no privilegiado (`spring`) para ejecutar el proceso de Java. Esto evita que, ante una potencial vulnerabilidad de ejecución remota de código (RCE) en la aplicación, el atacante obtenga acceso de root al host.
* **Dockerfile del Frontend (React):**
  * **Etapa 1 (Build):** Instala dependencias con `node:18-alpine` y genera la carpeta de distribución estática (`dist`) mediante `npm run build`, recibiendo parámetros de construcción (`ARG VITE_API_VENTAS_URL`).
  * **Etapa 2 (Run):** Copia los archivos estáticos a `nginx:alpine` y los sirve a través de Nginx. Incluye una configuración de redirección (`nginx.conf`) que mapea todas las rutas a `index.html` para dar soporte nativo al enrutador SPA.

### 3.2. Orquestación Local con Docker Compose
Para simular el entorno completo en local, el archivo [docker-compose.yml](file:///c:/Users/bg144/Downloads/proyecto.semestral4/proyecto%20semestral/proyecto%20semestral/docker-compose.yml) define:
* **Red interna:** Una red virtual tipo puente (`app-network`) que permite a los contenedores comunicarse usando sus nombres de servicio (ej. `DB_ENDPOINT=db`).
* **Volumen persistente:** `mysql_data` mapeado en `/var/lib/mysql` para evitar la pérdida de datos del contenedor de MySQL en cada reinicio.
* **Secuenciación de arranque:** Un `healthcheck` en MySQL (`mysqladmin ping`) asegura que las aplicaciones Spring Boot arranquen únicamente cuando la base de datos esté lista para aceptar conexiones (`condition: service_healthy`).

---

## 4. Pipeline de CI/CD (GitHub Actions) y Registro en Amazon ECR

El ciclo de despliegue se automatizó por completo mediante pipelines de GitHub Actions configurados en `.github/workflows/`:

1. **Compilación y Empaquetado:** Al realizar un push a `main`, el pipeline de GitHub Actions se activa, descarga el código fuente y configura el entorno de construcción Docker.
2. **Autenticación en AWS y Registro de Imágenes (Amazon ECR):**
   * El runner se conecta a AWS mediante `aws-actions/configure-aws-credentials` utilizando secretos de corta duración.
   * Realiza login en el registro privado de Amazon ECR de **Virginia (`us-east-1`)**.
   * Compila la imagen y la publica en el registro privado bajo las URIs correspondientes:
     * `545017861270.dkr.ecr.us-east-1.amazonaws.com/api_despachos:latest`
     * `545017861270.dkr.ecr.us-east-1.amazonaws.com/ventas_api:latest`
     * `545017861270.dkr.ecr.us-east-1.amazonaws.com/front_despacho:latest`
3. **Despliegue Continuo (CD) Seguro:**
   * El runner inicia una conexión SSH con las instancias EC2 en AWS utilizando una clave privada (`SSH_KEY_CITT`).
   * Para evitar almacenar claves estáticas de AWS en los servidores EC2 de producción, el runner genera un token temporal de lectura (`aws ecr get-login-password --region us-east-1`) y lo transmite de manera segura al servidor EC2 a través del túnel SSH para autenticar y realizar el `docker pull`.
   * Detiene el contenedor previo, elimina la imagen obsoleta e inicia el nuevo contenedor exponiendo los puertos requeridos.

---

## 5. Infraestructura en la Nube y Seguridad Básica

### 5.1. Arquitectura de Red y Recursos en AWS
La topología de red recomendada para producción en AWS garantiza alta seguridad y aislamiento de datos:
* **VPC (Virtual Private Cloud):** Una red virtual aislada en la región `us-east-1`.
* **Subredes Públicas:** Ubican el Frontend React (Nginx) y las APIs de backend en instancias EC2, permitiendo la comunicación HTTPS con los clientes de internet.
* **Subredes Privadas (Aislamiento de BD):** Aloja la base de datos relacional (ej. Amazon RDS MySQL). Esto impide que la base de datos tenga una IP pública y sea visible en Internet, permitiendo tráfico entrante únicamente desde las instancias de backend.
* **Grupos de Seguridad (Security Groups):** Actúan como firewalls virtuales a nivel de instancia:
  * El SG del Frontend solo permite tráfico HTTP/HTTPS (puertos 80/443) desde cualquier origen.
  * El SG de los Backends permite conexiones en los puertos `8080` y `8081` desde internet (para las llamadas API del cliente).
  * El SG de la Base de Datos solo acepta tráfico entrante en el puerto `3306` originado exclusivamente por las IPs privadas de los servidores de backend.
  * El acceso SSH (puerto 22) está restringido a IPs específicas o al pipeline de CI/CD.

### 5.2. Gestión de Secretos y Mínimo Privilegio
* **GitHub Secrets:** Almacena variables sensibles del pipeline de integración, como las credenciales de AWS (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`) y las llaves SSH de acceso a servidores.
* **Doppler:** Administrador centralizado de secretos que alimenta en caliente las variables de entorno de producción de las instancias de destino sin exponerlas en texto claro en repositorios.
* **Políticas IAM:** Las credenciales de despliegue de AWS se asocian a un rol de IAM específico que solo posee permisos restringidos para interactuar con Amazon ECR (lectura y escritura de imágenes), minimizando el impacto de seguridad ante un posible robo de credenciales.

---

## 6. Orquestación y Escalabilidad en Producción: ECS/EKS vs EC2 Manual

### 6.1. Clúster Aprovisionado y Balanceador de Carga en AWS
Como parte práctica de esta entrega, se ha configurado y desplegado de forma activa en la cuenta de AWS del estudiante en la región **us-east-1 (Norte de Virginia)**:
* **Clúster de Amazon ECS:** `proyecto-semestral-cluster`
* **Application Load Balancer (ALB):** `semestral-alb` (DNS: `semestral-alb-25385715.us-east-1.elb.amazonaws.com`)
* **Servicios de Fargate (Revisión v2):**
  * `ventas-service-v2`: Asociado al Target Group de Ventas (puerto 8080, ruta `/api/v1/ventas*`).
  * `despachos-service-v2`: Asociado al Target Group de Despachos (puerto 8081, ruta `/api/v1/despachos*`).
  * `front-despacho-service-v2`: Asociado al Target Group de Frontend (puerto 80, ruta `/*` por defecto).

Con esta configuración, el balanceador de carga actúa como punto de entrada único de la plataforma. Recibe todo el tráfico HTTP en el puerto 80 y lo distribuye dinámicamente a las tareas de Fargate basándose en reglas de enrutamiento por rutas. Esto elimina la necesidad de IPs estáticas o la exposición directa de los servidores de base.

Para ambientes de producción empresarial, el uso de despliegue manual en instancias EC2 presenta serias limitaciones operativas. Por ello, se fundamenta la adopción de servicios de orquestación administrados como **Amazon ECS (Elastic Container Service) con Fargate**:

1. **Operaciones del Servidor de Base (Serverless con Fargate):** Con EC2 manual, el administrador debe preocuparse de actualizar parches de seguridad del sistema operativo, configurar el daemon de Docker y gestionar el almacenamiento. ECS con AWS Fargate es serverless; elimina por completo la gestión del servidor físico o virtual, permitiendo enfocarse 100% en la aplicación.
2. **Escalabilidad Automática (Auto-scaling):** ECS gestiona de forma nativa políticas de auto-escalado horizontal muy rápidas y eficientes. Si la carga de la API de Ventas aumenta debido a un evento especial, ECS puede levantar contenedores adicionales en segundos y asociarlos al Application Load Balancer (ALB). En EC2 manual, esto requeriría aprovisionar servidores completos, configurar auto-scaling groups pesados y scripts de arranque.
3. **Alta Disponibilidad y Autocuración (Self-healing):** ECS monitorea constantemente el estado de salud de cada contenedor (`Target Groups`). Si un contenedor de Spring Boot se cuelga o agota su memoria, ECS lo elimina inmediatamente y despliega una instancia saludable en su lugar de forma transparente. En un esquema manual, el servicio quedaría inactivo hasta que un administrador del sistema ingrese por SSH y reinicie el contenedor.
4. **Despliegues sin Interrupciones (Zero-Downtime):** ECS soporta estrategias de despliegue tipo Rolling Updates de forma nativa. Al subir una nueva versión,ECS primero levanta los nuevos contenedores, comprueba su salud a través del balanceador y, una vez activos, destruye los contenedores antiguos de forma progresiva. En despliegues con scripts SSH manuales sobre una única instancia EC2, suele haber una ventana de caída de servicio (downtime) mientras se detiene, descarga y levanta el nuevo contenedor.

---

## 7. Conclusión

La solución propuesta resuelve las problemáticas tradicionales de despliegue manual mediante la estandarización de contenedores portables y seguros, la automatización total de pipelines de GitHub Actions integrados con Amazon ECR en Virginia, y el aislamiento de variables de entorno mediante Doppler. La adopción final de Amazon ECS con Fargate se consolida como la arquitectura ideal para llevar este proyecto a producción bajo un esquema moderno, robusto y altamente disponible.
