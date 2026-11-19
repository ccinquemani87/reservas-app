# Proyecto Integrador: reservas-app — Arquitectura e Infraestructura de Kubernetes con Helm

## 1. Resumen Ejecutivo y Ficha Técnica

El presente documento constituye el informe técnico consolidado del **Trabajo Final Integrador** para la asignatura *Contenedores, Docker y Orquestación con Kubernetes*. El proyecto documenta el diseño, empaquetado, despliegue y autoescalado de la infraestructura orientada a microservicios para la aplicación **`reservas-app`** (Variante B).

La aplicación es un sistema de reserva de salas por franja horaria estructurado en tres capas independientes: una interfaz web desarrollada en **React + Vite**, un backend de servicios **REST API en Node 20 + Express**, y un motor de almacenamiento de base de datos relacional en **MySQL 8.4**.

### Ficha Técnica de la Entrega

| Parámetro | Detalle de Infraestructura |
| :--- | :--- |
| **Repositorio de Código** | `reservas-app/` |
| **Etiqueta Git (Tag)** | `entrega-final` |
| **Variante de Proyecto** | Variante B (`reservas-app`: React + Express + MySQL) |
| **Empaquetado** | Helm Chart (`reservas-chart/`) |
| **Cantidad de Objetos Kubernetes** | 12 objetos oficiales empaquetados |
| **Ambiente de Despliegue** | Clúster Kubernetes con Ingress Controller y Metrics Server |

---

## 2. Arquitectura del Sistema de Tres Capas

La infraestructura de `reservas-app` se diseñó bajo el principio de desacoplamiento de capas, aislamiento de servicios y mínimo privilegio de acceso a los datos. La aplicación opera mediante el flujo de tráfico entrante a través del controlador Ingress hacia el clúster.

```
                           [ Cliente / Navegador ]
                                      |
                                      v (HTTP / reservas.local)
                         +--------------------------+
                         | Ingress Controller Nginx |
                         +--------------------------+
                               /              \
                   /api/reservas /              \ /
                                /                \
                               v                  v
                   +-------------------+   +-----------------------+
                   | Service (API)     |   | Service (Frontend)    |
                   | reservas-api:3000 |   | reservas-frontend:80  |
                   +-------------------+   +-----------------------+
                             |                         |
                             v                         v
                   +-------------------+   +-----------------------+
                   | Deployment (API)  |   | Deployment (Frontend) |
                   | Express (Node 20) |   | React + Nginx         |
                   | [HPA: 2 a 6 Pods] |   | [2 Réplicas]          |
                   +-------------------+   +-----------------------+
                             |
                             v (MySQL TCP 3306)
                   +-------------------+
                   | Service (DB)      |
                   | reservas-db:3306  |
                   +-------------------+
                             |
                             v
                   +-------------------+
                   | Deployment (DB)   |
                   | MySQL 8.4         |
                   +-------------------+
                             |
                             v (PV / Storage)
                   +-------------------+
                   | PVC 1Gi           |
                   | reservas-db-pvc   |
                   +-------------------+
```

### Flujo y Comunicación entre Componentes

1. **Capa de Entrada (Ingress Controller)**: Recibe las peticiones externas en el puerto 80 del host `reservas.local`. Rutea el tráfico `/api` directamente hacia el Service de la API y el tráfico raíz `/` hacia el Service del Frontend.
2. **Capa de Presentación (Frontend)**: Imagen multi-etapa optimizada basada en `nginxinc/nginx-unprivileged:alpine`. Sirve los archivos estáticos compilados de React y actúa como proxy inverso interno sin exponer directamente la API al exterior.
3. **Capa de Negocio (REST API)**: Proceso Express en Node 20 en ejecutor sin privilegios (`node`). Gestiona la lógica de negocios y la siembra/consulta de reservas hacia MySQL. Esta capa no conserva estado y escala dinámicamente según la demanda de CPU.
4. **Capa de Datos (Base de Datos MySQL)**: Instancia relacional MySQL 8.4 aislada dentro del clúster. No publica ningún puerto fuera de la red interna del clúster (`ClusterIP`) y almacena de forma persistente sus tablas mediante un `PersistentVolumeClaim`.

---

## 3. Inventario y Detalle de los 12 Objetos de Kubernetes

El proyecto integrador se encuentra empaquetado y gestionado a través de **12 objetos de Kubernetes**, declarados como plantillas dentro de `reservas-chart/templates/`.

### Tabla del Inventario de Objetos

| # | Tipo de Objeto | Nombre del Recurso | Función y Descripción Técnica |
| :---: | :--- | :--- | :--- |
| **1** | `Secret` | `reservas-db-secret` | Almacena credenciales sensibles de la base de datos (`MYSQL_ROOT_PASSWORD`, `MYSQL_USER`, `MYSQL_PASSWORD`) codificadas en Base64. |
| **2** | `PersistentVolumeClaim` | `reservas-db-pvc` | Solicita 1 GiB de almacenamiento persistente (`ReadWriteOnce`) para montar en `/var/lib/mysql` de la base de datos. |
| **3** | `ConfigMap` | `reservas-api-config` | Mantiene variables de entorno no sensibles de la API (`DB_HOST`, `DB_PORT`, `DB_NAME`, `PORT`). |
| **4** | `ConfigMap` | `reservas-frontend-config` | Almacena parámetros de ruteo y proxy para el frontend (`API_HOST`, `API_PORT`). |
| **5** | `Deployment` | `reservas-db` | Garantiza la ejecución de 1 réplica de MySQL 8.4 con volumen montado y credenciales inyectadas. |
| **6** | `Service` | `reservas-db` | Abstrae la dirección IP del pod de base de datos bajo un nombre DNS interno estable (`reservas-db:3306`). |
| **7** | `Deployment` | `reservas-api` | Gestiona las réplicas del backend de Node.js con solicitudes de recursos (`cpu: 100m`) para HPA. |
| **8** | `Service` | `reservas-api` | Provee un punto de acceso balanceado interno (`ClusterIP`) hacia las réplicas de la API en el puerto 3000. |
| **9** | `Deployment` | `reservas-frontend` | Mantiene 2 réplicas del servidor web Nginx sirviendo la aplicación web interactiva React. |
| **10** | `Service` | `reservas-frontend` | Publica la capa web de forma interna (`ClusterIP`) en el puerto 80 para ser alcanzada por Ingress. |
| **11** | `Ingress` | `reservas-app-ingress` | Define las reglas de ruteo HTTP por host (`reservas.local`) y path prefix hacia los Services correspondientes. |
| **12** | `HorizontalPodAutoscaler` | `reservas-api-hpa` | Regula automáticamente el número de réplicas del Deployment `reservas-api` entre 2 y 6 según el uso de CPU (umbral: 50%). |

---

## 4. Fundamentos Técnicos y Decisiones de Diseño

### 4.1. Seguridad y Gestión de Credenciales

* **Separación de Sensibilidad**: Los valores de configuración no sensibles se ubican en `ConfigMap` (`reservas-api-config` y `reservas-frontend-config`), mientras que las contraseñas del motor relacional se aíslan en `reservas-db-secret`.
* **Transformación y Mapeo de Variables**: El Secret `reservas-db-secret` define las claves que la imagen oficial de MySQL espera (`MYSQL_ROOT_PASSWORD`, `MYSQL_USER`, `MYSQL_PASSWORD`). Al inyectar estas credenciales en el Deployment de `reservas-api`, se utiliza la directiva `valueFrom.secretKeyRef` para renombrarlas al estándar que espera la API (`DB_USER`, `DB_PASSWORD`).
* **Principio de Mínimo Privilegio**: Ningún contenedor corre como `root`. Las imágenes creadas (`reservas-api` y `reservas-frontend`) ejecutan sus procesos bajo usuarios no privilegiados dedicados (`node` y `nginx` respectivamente). La base de datos MySQL no publica ningún puerto hacia el exterior (`NodePort` ni `LoadBalancer`), quedando restringida de manera exclusiva a la red interna del clúster.
* **Transparencia sobre Secrets**: Se documenta que un objeto `Secret` en Kubernetes aplica codificación Base64 y no cifrado criptográfico automático en reposo. Para un entorno de producción estricto, se requeriría habilitar la encriptación en reposo dentro de `etcd` o integrar gestores externos de secretos como HashiCorp Vault.

### 4.2. Persistencia del Dato y Estado de los Contenedores

* **Desacoplamiento de Persistencia**: Las capas de API REST y Frontend React son estrictamente *stateless* (sin estado). Si un pod de la API o del Frontend falla o se elimina, Kubernetes crea un reemplazo idéntico sin pérdida de datos.
* **Persistencia en MySQL (Stateful)**: Dado que los contenedores son efímeros, la eliminación de un pod de base de datos destruiría las reservas creadas de no contar con un volumen externo. La capa de datos implementa un `PersistentVolumeClaim` denominado `reservas-db-pvc` montado en el directorio `/var/lib/mysql`. Esto garantiza que, incluso si el pod `reservas-db` se destruye y el Deployment recrea la réplica, el nuevo pod se reconecta automáticamente al mismo PersistentVolume preservando todas las tablas e información de reservas.

### 4.3. Dependencias de Arranque y Ordenamiento en Kubernetes

En entornos como Docker Compose existía la directiva `depends_on: condition: service_healthy` para forzar la secuencia de arranque. En Kubernetes, dicha directiva no existe y la arquitectura debe ser naturalmente tolerante a fallas de ordenamiento:

* **Tolerancia a Conexión en API**: La API Express reintenta la conexión a MySQL durante un periodo inicial en caso de que el motor de base de datos se encuentre inicializando.
* **Resolución DNS en Nginx (Frontend)**: El servidor Nginx resuelve el nombre del upstream al iniciar. Si el Service de la API no existe antes de levantar el pod del frontend, Nginx arroja el error `host not found in upstream` y se detiene. Por esta razón, en Kubernetes y en la estructura del Helm Chart, los objetos de red (`Services`) se declaran e instancian **antes** que los controladores de carga (`Deployments`).

### 4.4. Autoescalado Horizontal (HPA) y Especificación de Recursos

* **Rol de `resources.requests`**: El controlador `HorizontalPodAutoscaler` calcula el porcentaje de consumo de la CPU utilizando la relación entre el uso real (obtenido desde `metrics-server`) y el valor declarado en `requests.cpu` del contenedor.
* **Solución al Estado `<unknown>`**: Si el Deployment `reservas-api` omite la sección `resources.requests`, el HPA carece de denominador matemático para la ecuación y la métrica informa indefinidamente `<unknown>`. Para habilitar el autoescalado, se especificó explícitamente:
  ```yaml
  resources:
    requests:
      cpu: "100m"
      memory: "64Mi"
    limits:
      cpu: "250m"
      memory: "128Mi"
  ```
* **Dinámica de Escalado**: Se fijó un piso de 2 réplicas y un techo de 6 réplicas para `reservas-api`, activando el escalado cuando el consumo medio de CPU supera el 50%. El escalado ascendente se ejecuta de manera acelerada al detectar picos de tráfico, mientras que el escalado descendente aplica una ventana de estabilización por defecto (5 minutos) para evitar oscilaciones innecesarias (*flapping*).

---

## 5. Empaquetado, Parametrización y Operación con Helm

### 5.1. Estructura del Chart (`reservas-chart/`)

El conjunto completo de recursos se consolidó en la estructura estándar de Helm:

```
reservas-chart/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── configmap-api.yaml
    ├── configmap-frontend.yaml
    ├── hpa-api.yaml
    ├── ingress.yaml
    ├── pvc-db.yaml
    ├── reservas-api.yaml
    ├── reservas-db.yaml
    ├── reservas-frontend.yaml
    └── secret-db.yaml
```

### 5.2. Parametrización en `values.yaml`

Para garantizar que la release sea reutilizable en entornos de desarrollo, staging y producción sin modificar las plantillas YAML, se centralizaron las variables en `values.yaml`:

```yaml
frontend:
  replicaCount: 2
  image:
    repository: ccinquemani87/reservas-frontend
    tag: v1

api:
  replicaCount: 2
  image:
    repository: ccinquemani87/reservas-api
    tag: v1

db:
  image:
    repository: mysql
    tag: "8.4"

hpa:
  minReplicas: 2
  maxReplicas: 6
  targetCPU: 50
```

### 5.3. Guía de Operaciones del Ciclo de Vida con Helm

#### 1. Validación e Inspección de Manifiestos
Antes de instalar, se simula el renderizado de los 12 manifiestos procesando las variables de `values.yaml`:

```bash
helm template reservas-app ./reservas-chart
```

#### 2. Instalación de la Release
Se despliega la release en el namespace objetivo:

```bash
helm install reservas-app ./reservas-chart -n reservas-app
```

#### 3. Verificación de Recursos Desplegados
Se comprueba que los 12 recursos hayan sido creados correctamente bajo la gestión de Helm:

```bash
kubectl get all,cm,secret,pvc,ingress,hpa -n reservas-app
```

#### 4. Actualización Progresiva (Upgrade)
Se simula la actualización de la versión del backend modificando la etiqueta de la imagen a `v2` en tiempo de ejecución:

```bash
helm upgrade reservas-app ./reservas-chart --set api.image.tag=v2 -n reservas-app
```

#### 5. Inspección del Historial de Releases
Se consulta el registro de revisiones gestionado por Helm:

```bash
helm history reservas-app -n reservas-app
```

#### 6. Reversión de Cambios (Rollback)
Ante cualquier falla en una actualización, se restablece el clúster a la Revisión 1:

```bash
helm rollback reservas-app 1 -n reservas-app
```

---

## 6. Conclusión

El proyecto integrador **`reservas-app`** demuestra el ciclo completo de modernización e ingeniería de infraestructura cloud-native: desde la contenerización multi-etapa y el aislamiento en entornos aislados de Docker, hasta el despliegue orquestado, tolerante a fallos, autoescalable y empaquetado profesionalmente con Helm en Kubernetes.

Los 12 objetos de infraestructura cumplen estrictamente con los estándares de seguridad, alta disponibilidad, persistencia de datos y mantenibilidad exigidos para entornos de producción.
